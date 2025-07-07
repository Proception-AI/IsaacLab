# LeRobot Dataset File Handler Design Document

## Overview

The LeRobot Dataset File Handler (`LeRobotDatasetFileHandler`) is a configuration-driven system for automatically extracting and recording episode data from Isaac Lab environments to the LeRobot dataset format. It provides a seamless bridge between Isaac Lab's manager-based environments and the HuggingFace LeRobot ecosystem, enabling efficient dataset creation for Vision-Language-Action (VLA) model training.

## Dependencies

- `lerobot` for dataset interfacing: https://github.com/huggingface/lerobot

https://github.com/huggingface/lerobot/issues/1398

## Architecture

### Core Components

1. **Configuration System** (`LeRobotDatasetCfg`)
   - Defines which observations to record and how to organize them
   - Supports both regular observations and state observations
   - Configurable task descriptions

2. **Feature Extraction Engine**
   - Automatically analyzes environment observation and action managers
   - Handles nested observation structures with group-based access
   - Detects and processes video/image features automatically

3. **Data Conversion Pipeline**
   - Converts Isaac Lab episode data to LeRobot format
   - Handles tensor shape transformations and data type conversions
   - Manages video/image format conversions ([B, H, W, C] → [C, H, W])

4. **Dataset Management**
   - Creates and manages LeRobot dataset files
   - Handles episode writing and metadata management
   - Provides efficient storage with MP4 videos and Parquet files

## Configuration System

### RecorderManagerBaseCfg Structure

The LeRobot dataset configuration is now integrated into the `RecorderManagerBaseCfg` class:

```python
@configclass
class RecorderManagerBaseCfg:
    # Standard recorder configuration
    dataset_file_handler_class_type: type = HDF5DatasetFileHandler
    dataset_export_dir_path: str = "/tmp/isaaclab/logs"
    dataset_filename: str = "dataset"
    dataset_export_mode: DatasetExportMode = DatasetExportMode.EXPORT_ALL
    export_in_record_pre_reset: bool = True
    
    # LeRobot dataset specific configuration
    observation_keys_to_record: Optional[List[tuple[str, str]]] = None
    """List of (group_name, observation_key) tuples to record as regular observations.
    
    These will be saved as "observation.{obs_key}" in the LeRobot format.
    Example: [("policy", "joint_pos"), ("policy", "camera_rgb"), ("critic", "joint_vel")]
    """

    state_observation_keys: Optional[List[tuple[str, str]]] = None
    """List of (group_name, observation_key) tuples that should be treated as state observations.
    
    These will be combined and saved as "observation.state" in the LeRobot format.
    Example: [("policy", "joint_pos"), ("policy", "joint_vel")]
    """

    task_description: Optional[str] = None
    """Task description for the LeRobot dataset.
    
    This description will be used for all episodes in the dataset.
    Example: "Stack the red cube on top of the blue cube"
    """
```

### Configuration Patterns

#### Basic Configuration
```python
env.cfg.recorders.observation_keys_to_record = [
    ("policy", "joint_pos"), 
    ("policy", "camera_rgb")
]
env.cfg.recorders.state_observation_keys = [
    ("policy", "joint_vel")
]
env.cfg.recorders.task_description = "Stack the red cube on top of the blue cube"
```

#### Multi-Group Observations
```python
env.cfg.recorders.observation_keys_to_record = [
    ("policy", "joint_pos"),      # From policy group
    ("policy", "camera_rgb"),     # From policy group
    ("critic", "joint_vel")       # From critic group
]
```

#### Video/Image Support
```python
env.cfg.recorders.observation_keys_to_record = [
    ("policy", "camera_rgb")      # Automatically detected as video
]
# Handles [B, H, W, C] format and converts to [C, H, W] for LeRobot
```

## Feature Extraction Process

### 1. Environment Analysis
The handler automatically analyzes the environment's structure:

```python
def _extract_features_from_env(self, env) -> Dict[str, Dict]:
    features = {}
    
    # Extract action features
    features.update(self._extract_action_features(env))
    
    # Extract observation features
    features.update(self._extract_observation_features(env))
    
    # Add annotation features
    features.update(self._extract_annotation_features(env))
    
    return features
```

### 2. Action Feature Extraction
```python
def _extract_action_features(self, env) -> Dict[str, Dict]:
    return {
        "action": {
            "dtype": "float32",
            "shape": (env.action_manager.total_action_dim,),
            "names": None
        }
    }
```

### 3. Observation Feature Extraction
The system processes observations based on configuration:

- **Regular Observations**: Saved as `observation.{key}`
- **State Observations**: Combined into `observation.state`
- **Video Detection**: Automatically detects 4D tensors in [B, H, W, C] format

### 4. Video/Image Processing
```python
def _is_video_feature(self, tensor: torch.Tensor) -> bool:
    # Check if tensor has exactly 4 dimensions
    if tensor.ndim == 4:
        # Validate [B, H, W, C] format
        if (tensor.shape[1] > 1 and tensor.shape[2] > 1 and tensor.shape[3] <= 4):
            return True
    return False
```

## Data Flow

### Episode Recording Process

1. **Episode Creation**
   ```python
   handler = LeRobotDatasetFileHandler()
   handler.create("dataset.lerobot", env_name="my_env", env=env)
   ```

2. **Feature Schema Generation**
   - Analyzes environment observation and action managers
   - Creates LeRobot-compatible feature specifications
   - Validates configuration against available observations

3. **Episode Writing**
   ```python
   handler.write_episode(episode_data)
   ```

4. **Frame-by-Frame Processing**
   ```python
   def _convert_and_save_episode(self, episode: EpisodeData):
       for frame_idx in range(num_frames):
           frame_data = {}
           
           # Process actions
           frame_data.update(self._process_actions(frame_action))
           
           # Process observations
           frame_obs = self._extract_frame_observations(obs_dict, frame_idx)
           frame_data.update(self._process_observations(frame_obs))
           
           # Add to dataset
           self._dataset.add_frame(frame_data, task)
   ```

## Integration with Isaac Lab

### Environment Configuration
The handler integrates seamlessly with Isaac Lab's manager-based environments:

```python
# In environment configuration
env_cfg.recorders.observation_keys_to_record = [("policy", "camera_rgb")]
env_cfg.recorders.state_observation_keys = [("policy", "joint_pos")]
env_cfg.recorders.task_description = "Custom task"

# In recorder configuration
env_cfg.recorders.dataset_file_handler_class_type = LeRobotDatasetFileHandler
```

### Recording Script Integration
The `record_demos.py` script automatically detects LeRobot format:

```python
# Configure dataset format based on file extension
use_lerobot_format = args_cli.dataset_file.endswith('.lerobot')

if use_lerobot_format:
    env_cfg.recorders.dataset_file_handler_class_type = LeRobotDatasetFileHandler
else:
    env_cfg.recorders.dataset_file_handler_class_type = HDF5DatasetFileHandler
```

## Dataset Structure

### LeRobot Format Organization
```
dataset.lerobot/
├── dataset_info.json          # HuggingFace dataset metadata
├── state.json                 # Dataset state information
├── data/                      # Parquet files with episode data
│   ├── train-00000-of-00001.parquet
│   └── ...
├── videos/                    # Video files for camera observations
│   ├── episode_000000/
│   │   ├── front.mp4
│   │   ├── wrist.mp4
│   │   └── ...
│   └── episode_000001/
│       └── ...
└── meta/                      # Additional metadata
    └── info.json              # Isaac Lab specific metadata
```

### Feature Naming Conventions
- **Camera observations**: `observation.images.{camera_position}`
- **Robot state**: `observation.state`
- **Regular observations**: `observation.{obs_key}`
- **Actions**: `action`
- **Episode metadata**: `episode_index`, `frame_index`, `timestamp`, `task`

## Error Handling and Validation

### Configuration Validation
```python
# Validate that required configuration exists
if not observation_keys_to_record:
    raise ValueError(
        "RecorderManagerBaseCfg must have observation_keys_to_record configured. "
        "Please set observation_keys_to_record with format: [('group_name', 'observation_key'), ...]"
    )

if not state_observation_keys:
    raise ValueError(
        "RecorderManagerBaseCfg must have state_observation_keys configured. "
        "Please set state_observation_keys with format: [('group_name', 'observation_key'), ...]"
    )

# Validate observation groups and keys
if group_name not in obs_sample:
    available_groups = list(obs_sample.keys())
    raise ValueError(f"Observation group '{group_name}' not found. Available groups: {available_groups}")
```

### Video Format Validation
```python
def _is_video_feature(self, tensor: torch.Tensor) -> bool:
    if tensor.ndim == 4:
        if not (tensor.shape[1] > 1 and tensor.shape[2] > 1 and tensor.shape[3] <= 4):
            raise ValueError(
                f"Image data must be in [B, H, W, C] format, but got shape {tensor.shape}. "
                f"Expected format: batch_size > 0, height > 1, width > 1, channels <= 4"
            )
        return True
    return False
```

## Performance Considerations

### Memory Management
- Efficient tensor processing with minimal memory overhead
- Automatic cleanup of temporary data structures
- Background video writing for large datasets

### Storage Optimization
- MP4 compression for video observations
- Parquet format for efficient tabular data storage
- Incremental episode writing to avoid memory accumulation

## Usage Examples

### Basic Recording
```python
# Configure environment
env_cfg.lerobot_dataset.observation_keys_to_record = [("policy", "camera_rgb")]
env_cfg.lerobot_dataset.state_observation_keys = [("policy", "joint_pos")]
env_cfg.lerobot_dataset.task_description = "Stack cubes"

# Record dataset
./isaaclab.sh -p scripts/tools/record_demos.py \
    --task Isaac-Stack-Cube-Franka-IK-Rel-v0 \
    --teleop_device spacemouse \
    --dataset_file ./datasets/demo.lerobot
```

### Advanced Configuration
```python
# Multi-camera setup
env_cfg.lerobot_dataset.observation_keys_to_record = [
    ("policy", "camera_rgb"),      # Main camera
    ("policy", "camera_depth"),    # Depth camera
    ("policy", "wrist_camera"),    # Wrist camera
    ("policy", "end_effector_pos") # End effector position
]

# Comprehensive state representation
env_cfg.lerobot_dataset.state_observation_keys = [
    ("policy", "joint_pos"),       # Joint positions
    ("policy", "joint_vel"),       # Joint velocities
    ("policy", "gripper_state")    # Gripper state
]
```
