# object-detection-model

## Preaparation for train

### Prepare train & test images
- use labelImg as an annotation tool and save annotation files using Pascal Voc Xml format
```bash
pip install labelImg
# run labelImg
labelImg
```
- create a folder 'images' to save images and a folder 'annotations' to save annotation xml files separately
- convert xml annotation files to a csv file using data_generation/xml_to_csv.py
  ```bash
  cd data_generation
  python xml_to_csv.py --annot_dir ../data/kimchi/train/annotations --out_csv_path ../data/kimchi/train_labels.csv
  python xml_to_csv.py --annot_dir ../data/kimchi/test/annotations --out_csv_path ../data/kimchi/test_labels.csv
  ```
- convert csv files for train and test to tfrecord files using generate_tfrecord.py
  ```bash
  # create tfrecords for train images
  python generate_tfrecord.py --path_to_images ../data/kimchi/train/images --path_to_annot ../data/kimchi/train_labels.csv --path_to_label_map ../models/kimchi_labelmap.pbtxt --path_to_save_tfrecords ../data/kimchi/train.record
  # create tfrecords for test images
  python generate_tfrecord.py --path_to_images ../data/kimchi/test/images --path_to_annot ../data/kimchi/test_labels.csv --path_to_label_map ../models/kimchi_labelmap.pbtxt --path_to_save_tfrecords ../data/kimchi/val.record
  ```
### Prepare pretrained model
```bash
cd models/
# download the mobilenet_v2 model
wget http://download.tensorflow.org/models/object_detection/tf2/20200711/ssd_mobilenet_v2_320x320_coco17_tpu-8.tar.gz
# extract the downloaded file
tar -xzvf ssd_mobilenet_v2_320x320_coco17_tpu-8.tar.gz
```

### Model files
- .config file(Examples)
  ```config
  # SSD with Mobilenet v2

  model {
    ssd {
      inplace_batchnorm_update: true
      freeze_batchnorm: false
      num_classes: 1
      box_coder {
        faster_rcnn_box_coder {
          y_scale: 10.0
          x_scale: 10.0
          height_scale: 5.0
          width_scale: 5.0
        }
      }
      matcher {
        argmax_matcher {
          matched_threshold: 0.4
          unmatched_threshold: 0.4
          ignore_thresholds: false
          negatives_lower_than_unmatched: true
          force_match_for_each_row: true
          use_matmul_gather: true
        }
      }
      similarity_calculator {
        iou_similarity {
        }
      }
      encode_background_as_zeros: true
      anchor_generator {
        ssd_anchor_generator {
          num_layers: 6
          min_scale: 0.15
          max_scale: 0.95
          aspect_ratios: 1.0
          aspect_ratios: 2.0
          aspect_ratios: 0.5
        }
      }
      image_resizer {
        fixed_shape_resizer {
          height: 300
          width: 300
        }
      }
      box_predictor {
        convolutional_box_predictor {
          min_depth: 0
          max_depth: 0
          num_layers_before_predictor: 0
          use_dropout: false
          dropout_keep_probability: 0.8
          kernel_size: 1
          box_code_size: 4
          apply_sigmoid_to_scores: false
          class_prediction_bias_init: -4.6
          conv_hyperparams {
            activation: RELU_6,
            regularizer {
              l2_regularizer {
                weight: 0.00004
              }
            }
            initializer {
              random_normal_initializer {
                stddev: 0.01
                mean: 0.0
              }
            }
            batch_norm {
              train: true,
              scale: true,
              center: true,
              decay: 0.97,
              epsilon: 0.001,
            }
          }
        }
      }
      feature_extractor {
        type: 'ssd_mobilenet_v2_keras'
        min_depth: 16
        depth_multiplier: 1.0
        conv_hyperparams {
          activation: RELU_6,
          regularizer {
            l2_regularizer {
              weight: 0.00004
            }
          }
          initializer {
            truncated_normal_initializer {
              stddev: 0.03
              mean: 0.0
            }
          }
          batch_norm {
            train: true,
            scale: true,
            center: true,
            decay: 0.97,
            epsilon: 0.001,
          }
        }
        override_base_feature_extractor_hyperparams: true
      }
      loss {
        classification_loss {
          weighted_sigmoid_focal {
            alpha: 0.75,
            gamma: 2.0
          }
        }
        localization_loss {
          weighted_smooth_l1 {
            delta: 1.0
          }
        }
        classification_weight: 1.0
        localization_weight: 1.0
      }
      normalize_loss_by_num_matches: true
      normalize_loc_loss_by_codesize: true
      post_processing {
        batch_non_max_suppression {
          score_threshold: 0.001
          iou_threshold: 0.4
          max_detections_per_class: 100
          max_total_detections: 100
        }
        score_converter: SIGMOID
      }
    }
  }

  train_config: {
    fine_tune_checkpoint_version: V2
    fine_tune_checkpoint: "../models/ssd_mobilenet_v2_320x320_coco17_tpu-8/checkpoint/ckpt-0"
    fine_tune_checkpoint_type: "detection"
    batch_size: 16
    sync_replicas: true
    startup_delay_steps: 0
    replicas_to_aggregate: 8
    num_steps: 3000
    data_augmentation_options {
      random_horizontal_flip {
      }
    }
    data_augmentation_options {
      ssd_random_crop {
      }
    }
    optimizer {
      momentum_optimizer: {
        learning_rate: {
          cosine_decay_learning_rate {
            learning_rate_base: 0.025
            total_steps: 3000
            warmup_learning_rate: 0.005
            warmup_steps: 100
          }
        }
        momentum_optimizer_value: 0.9
      }
      use_moving_average: false
    }
    max_number_of_boxes: 100
    unpad_groundtruth_tensors: false
  }

  train_input_reader: {
    label_map_path: "../models/kimchi_labelmap.pbtxt"
    tf_record_input_reader {
      input_path: "../data/kimchi/train.record"
    }
  }

  eval_config: {
    metrics_set: "coco_detection_metrics"
    use_moving_averages: false
  }

  eval_input_reader: {
    label_map_path: "../models/kimchi_labelmap.pbtxt"
    shuffle: false
    num_epochs: 1
    tf_record_input_reader {
      input_path: "../data/kimchi/val.record"
    }
  }
  ```
- .pbtxt
  ```text
  item {
  id: 1
  name: 'kimchi'
  }

  ```

## Train
- train command
  ```bash
  out_dir=../models/ssd_mobilenet_v2_kimchi/
  mkdir -p $out_dir
  python model_main_tf2.py --alsologtostderr --model_dir=$out_dir --checkpoint_every_n=500  \
                         --pipeline_config_path=../models/ssd_mobilenet_v2_kimchi.config \
                         --eval_on_train_data 2>&1 | tee $out_dir/train.log
  ```
- tensorboard
  ```bash
  tensorboard --logdir=models/ssd_mobilenet_v2_kimchi
  ```

## Inference Test
```bash
python detect_objects.py --threshold 0.5 --model_path ../models/ssd_mobilenet_v2_kimchi/exported_model/saved_model --path_to_labelmap ../models/kimchi_labelmap.pbtxt --images_dir /home/jinwon/Documents/github/object-detection-model/data/kimchi/test/images --save_output
```
