# Fine Tuning a Model


## Introduction

This page outlines the steps to fine-tune an existing pre-trained model with T5X
on common downstream tasks defined with [SeqIO](https://github.com/google/seqio/blob/main/README.md). This is one of the
simplest and most common use cases of T5X. If you're new to T5X, this tutorial
is the recommended starting point.

## Overview

Fine-tuning a model with T5X consists of the following steps:

1.  Choose the pre-trained model to fine-tune.
2.  Choose the SeqIO Task/Mixture to fine-tune the model on.
3.  Write a Gin file that configures the pre-trained model, SeqIO Task/Mixture
    and other details of your fine-tuning run.
4.  Launch your experiment locally or on XManager.
5.  Monitor your experiment and parse metrics.

These steps are explained in detail in the following sections. An example run
that fine-tunes a T5-small checkpoint on GLUE benchmark is also showcased.

## Step 1: Choose a pre-trained model

To use a pre-trained model, you need a Gin config file that defines the model
params, and the model checkpoint to load from. For your convenience, TensorFlow
checkpoints and Gin configs for common T5 pre-trained models have been made
available for use in T5X. Following is a list of these pre-trained models and
their Gin and checkpoint locations.

+   All checkpoints:
    [`gs://t5-data/pretrained_models/t5x/`](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/)
+   All Gin files:
    [`t5x/configs/models/`](https://github.com/google-research/t5x/tree/main/t5x/configs/)

### Public Research Models

#### T5 Checkpoints

These are the checkpoints used in the paper [Exploring the Limits of Transfer
Learning with a Unified Text-to-Text
Transformer](https://arxiv.org/abs/1910.10683). They are encoder-decoder models
trained on [C4](https://www.tensorflow.org/datasets/catalog/c4) with a denoising
objective.

Model    | Gin File Location                                                              | Checkpoint Location
-------- | ------------------------------------------------------------------------------ | -------------------
T5 Small | [t5_small.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_0/small.gin) | [gs://t5-data/pretrained_models/t5x/t5_small/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_small)
T5 Base  | [t5_base.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_0/base.gin)   | [gs://t5-data/pretrained_models/t5x/t5_base/checkpoint_999900](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_base)
T5 Large | [t5_large.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_0/large.gin) | [gs://t5-data/pretrained_models/t5x/t5_large/checkpoint_1000700](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_large)
T5 3B    | [t5_3B.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_0/3B.gin)       | [gs://t5-data/pretrained_models/t5x/t5_3B/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_3B)
T5 11B   | [t5_11B.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_0/11B.gin)     | [gs://t5-data/pretrained_models/t5x/t5_11B/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_11B)

#### T5 1.1 Checkpoints

These are similar to the models from [Exploring the Limits of Transfer Learning
with a Unified Text-to-Text Transformer](https://arxiv.org/abs/1910.10683), but
with the following improvements:

*   GEGLU activation in feed-forward hidden layer, rather than ReLU - see
    https://arxiv.org/abs/2002.05202 .
*   Dropout was turned off in pre-training (quality win). Dropout should be
    re-enabled during fine-tuning.
*   Pre-trained on C4 only without mixing in the downstream tasks.
*   no parameter sharing between embedding and classifier layer
*   "xl" and "xxl" replace "3B" and "11B". The model shapes are a bit
    different - larger d_model and smaller num_heads and d_ff.

For English-language, sequence-to-sequence-style tasks (ones where the goal is
to map from an input text sequence to a target sequence) these are usually the
best models to fine-tune.

Model        | Gin File Location                                                                  | Checkpoint Location
------------ | ---------------------------------------------------------------------------------- | -------------------
T5 1.1 Small | [t5_1_1/small.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_1/small.gin) | [gs://t5-data/pretrained_models/t5x/t5_1_1_small/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_small)
T5 1.1 Base  | [t5_1_1/base.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_1/base.gin)   | [gs://t5-data/pretrained_models/t5x/t5_1_1_base/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_base)
T5 1.1 Large | [t5_1_1_large.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_1/large.gin) | [gs://t5-data/pretrained_models/t5x/t5_1_1_large/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_large)
T5 1.1 XL    | [t5_1_1_xl.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_1/xl.gin)       | [gs://t5-data/pretrained_models/t5x/t5_1_1_xl/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_xl)
T5 1.1 XXL   | [t5_1_1_xxl.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_1/xxl.gin)     | [gs://t5-data/pretrained_models/t5x/t5_1_1_xxl/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_xxl)

#### T5 1.1 LM-Adapted Checkpoints

These "LM-adapted" models are initialized from T5 1.1 (above) and trained for
an additional 100K steps on the LM objective discussed in the [T5
paper](https://arxiv.org/abs/1910.10683). This adaptation improves the ability
of the model to be used for [prompt tuning](https://arxiv.org/abs/2104.08691).
These checkpoints were also used within the BigScience
[T0](https://arxiv.org/abs/2110.08207) project.

Model        | Gin File Location                                                                                                   | Checkpoint Location
------------ | ------------------------------------------------------------------------------------------------------------------- | -------------------
T5 1.1 LM-100K Small | [t5_1_1_small.gin](https://github.com/google-research/t5x/tree/main/t5x/google/examples/flaxformer_t5/configs/models/t5_1_1_small.gin) | [t5_1_1_lm100k_small/checkpoint_1100000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_lm100k_small)
T5 1.1 LM-100K Base  | [t5_1_1_base.gin](https://github.com/google-research/t5x/tree/main/t5x/google/examples/flaxformer_t5/configs/models/t5_1_1_base.gin)   | [t5_1_1_lm100k_base/checkpoint_1100000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_lm100k_base)
T5 1.1 LM-100K Large | [t5_1_1_large.gin](https://github.com/google-research/t5x/tree/main/t5x/google/examples/flaxformer_t5/configs/models/t5_1_1_large.gin) | [t5_1_1_lm100k_large/checkpoint_1100000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_lm100k_large)
T5 1.1 LM-100K XL    | [t5_1_1_xl.gin](https://github.com/google-research/t5x/tree/main/t5x/google/examples/flaxformer_t5/configs/models/t5_1_1_xl.gin)       | [t5_1_1_lm100k_xl/checkpoint_1100000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_lm100k_xl)
T5 1.1 LM-100K XXL   | [t5_1_1_xxl.gin](https://github.com/google-research/t5x/tree/main/t5x/google/examples/flaxformer_t5/configs/models/t5_1_1_xxl.gin)     | [t5_1_1_lm100k_xxl/checkpoint_1100000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_1_1_lm100k_xxl)

#### MT5 Checkpoints

These are the checkpoints used in the paper
[mT5: A Massively Multilingual Pre-trained Text-to-Text Transformer](https://aclanthology.org/2021.naacl-main.41/).
They are encoder-decoder models trained on
[multilingual C4](https://www.tensorflow.org/datasets/catalog/c4#c4multilingual)
with a denoising objective. These are the best checkpoints to fine-tune for
non-English sequence-to-sequence tasks.

Model     | Gin File Location                                                            | Checkpoint Location
--------- | ---------------------------------------------------------------------------- | -------------------
MT5 Small | [mt5/small.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/mt5/small.gin) | [gs://t5-data/pretrained_models/t5x/mt5_small/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/mt5_small)
MT5 Base  | [mt5/base.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/mt5/base.gin)   | [gs://t5-data/pretrained_models/t5x/mt5_base/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/mt5_base)
MT5 Large | [mt5/large.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/mt5/large.gin) | [gs://t5-data/pretrained_models/t5x/mt5_large/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/mt5_large)
MT5 XL    | [mt5/xl.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/mt5/xl.gin)       | [gs://t5-data/pretrained_models/t5x/mt5_xl/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/mt5_xl)
MT5 XXL   | [mt5/xxl.gin](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/mt5/xxl.gin)     | [gs://t5-data/pretrained_models/t5x/mt5_xxl/checkpoint_1000000](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/mt5_xxl)


For the example run, you will use the T5 Small model. The Gin file for this
model is located at
[`/t5x/examples/t5/t5_1_0/small.gin`](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_0/small.gin),
and the checkpoint is located at
[`gs://t5-data/pretrained_models/t5x/t5_small`](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/t5x/t5_small).

## Step 2: Choose a SeqIO Task/Mixture

A SeqIO Task encapsulates the data source, the preprocessing logic to be
performed on the data before querying the model, the postprocessing logic to be
performed on model outputs, and the metrics to be computed given the
postprocessed outputs and targets. A SeqIO Mixture denotes a collection of Tasks
and enables fine-tuning a model on multiple Tasks simultaneously.

### Standard Tasks

Many common datasets and benchmarks, e.g. [GLUE](https://gluebenchmark.com/),
[SuperGLUE](https://super.gluebenchmark.com/),
[WMT](https://www.tensorflow.org/datasets/catalog/wmt_t2t_translate),
[SQUAD](https://rajpurkar.github.io/SQuAD-explorer/),
[CNN/Daily Mail](https://github.com/abisee/cnn-dailymail), etc. have been
implemented as SeqIO Tasks/Mixtures and can be used directly. These
Tasks/Mixtures are defined in
[`third_party/py/t5/data/tasks.py`](https://github.com/google-research/text-to-text-transfer-transformer/tree/main/t5/data/tasks.py)
and
[`third_party/py/t5/data/mixtures.py`](https://github.com/google-research/text-to-text-transfer-transformer/tree/main/t5/data/mixtures.py).

For the example run, you will fine-tune the model on the GLUE benchmark, which
has been implemented as the
[`glue_v002_proportional`](https://github.com/google-research/text-to-text-transfer-transformer/tree/main/t5/data/mixtures.py?l=52&rcl=367537028)
Mixture.

### Custom Tasks

It is also possible to define your own custom task. See the
[SeqIO documentation](https://github.com/google/seqio/blob/main/README.md) for how to do this. As a note, Tasks defined
using the
[old T5 codebase](https://github.com/google-research/text-to-text-transfer-transformer/tree/main/t5/data/dataset_providers.py)
may also be used by T5X. If using a custom Task, you will need to follow the
instructions in the [Advanced Topics section](#custom-t5x-binaries) at the end
of this tutorial to make sure the module containing your task is included.

When defining a custom task, you have the option to cache it on disk before
fine-tuning. The instructions for this are
[here](https://github.com/google/seqio/blob/main/README.md#optional-offline-caching). Caching may improve performance for
tasks with expensive pre-processing. By default, T5X expects tasks to be cached.
To finetune on a task that has not been cached, set
`--gin.USE_CACHED_TASKS=False`.

## Step 3: Write a Gin Config

After choosing the pre-trained model and SeqIO Task/Mixture for your run, the
next step is to configure your run using Gin. If you're not familiar with Gin,
reading the [T5X Gin Primer](gin.md) is recommended.

T5X provides a Gin file that configures the T5X trainer for fine-tuning (located
at
[`p5x/configs/runs/finetune.gin`](https://github.com/google-research/t5x/tree/main/t5x/configs/runs/finetune.gin)),
and expects a few params from you. These params can be specified in a separate
Gin file, or via commandline flags. Following are the required params:

+   `INITIAL_CHECKPOINT_PATH`: This is the path to the pre-trained checkpoint
    (from Step 1). For the example run, set this to
    `'gs://t5-data/pretrained_models/t5x/t5_small/checkpoint_1000000'`.
+   `TRAIN_STEPS`: Number of fine-tuning steps. This includes the number of
    steps that the model was pre-trained for, so make sure to add the step
    number from the `INITIAL_CHECKPOINT_PATH`. For the example run, to fine-tune
    for `500_000` steps, set this to `1_500_000`, since the initial checkpoint
    is the `1_000_000`th step.
+   `MIXTURE_OR_TASK_NAME`: This is the SeqIO Task or Mixture name to run (from
    Step 2). For the example run, set this to `'glue_v002_proportional'`.
+   `TASK_FEATURE_LENGTHS`: This is a dict mapping feature key to maximum int
    length for that feature. After preprocessing, features are truncated to the
    provided value. For the example run, set this to `{'inputs': 512, 'targets':
    84}`.
+   `MODEL_DIR`: A path to write fine-tuned checkpoints to. When launching using
    XManager, this path is automatically set and can be accessed from the
    XManager Artifacts page. When running locally using Blaze, you can
    explicitly pass a directory using a flag. Launch commands are provided in
    the next step.
+   `LOSS_NORMALIZING_FACTOR`: When fine-tuning a model that was pre-trained
    using Mesh Tensorflow (e.g. the public T5 / mT5 / ByT5 models), this should
    be set to `pretraining batch_size` * `pretrained target_token_length`. For
    T5 and T5.1.1: `2048 * 114`. For mT5: `1024 * 229`. For ByT5: `1024 * 189`.
    For MUM Base/Large/XL: `1024 * 256`. For MUM XXL: `1024 * 229`. For MUM Ace:
    `4096 * 229`.

In addition to the above params, you will need to include
[`finetune.gin`](https://github.com/google-research/t5x/tree/main/t5x/configs/runs/finetune.gin)
and the Gin file for the pre-trained model, which for the example run is
[`t5_1_1/small.gin`](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_1/small.gin).

```gin
include 't5x/configs/runs/finetune.gin'
include 't5x/examples/t5/t5_1_1/small.gin'
```

You will also need to import the Python module(s) that register SeqIO Tasks and
Mixtures used in your run. For the example run, we add `import t5.data.mixtures`
since it is where 'glue_v002_proportional' is registered.


Finally, your Gin file should look like this:

```gin
include 't5x/configs/runs/finetune.gin'
include 't5x/examples/t5/t5_1_1/small.gin'

# Register necessary SeqIO Tasks/Mixtures.
import t5.data.mixtures

INITIAL_CHECKPOINT_PATH = 'gs://t5-data/pretrained_models/t5x/t5_1_1_base/checkpoint_1000000'
TRAIN_STEPS = 1_020_000  # includes 1_000_000 pretrain steps
MIXTURE_OR_TASK_NAME = "wmt_t2t_ende_v003"
TASK_FEATURE_LENGTHS = {"inputs": 256, "targets": 256}
LOSS_NORMALIZING_FACTOR = 233472
```

See
[`t5x/examples/t5/t5_1_1/examples/small_wmt_finetune.gin`](https://github.com/google-research/t5x/tree/main/t5x/examples/t5/t5_1_1/examples/small_wmt_finetune.gin)
for this example.


## Step 4: Launch your experiment

To launch your experiment locally (for debugging only; larger checkpoints may
cause issues), run the following on commandline:

```sh
MODEL_DIR="/tmp/finetune-model/"
python -m t5x.train \
  --gin_file=t5x/examples/t5/t5_0_0/examples/small_glue_finetune.gin \
  --gin.MODEL_DIR=\"${MODEL_DIR}\" \
  --alsologtostderr
```

Note that multiple comma-separated paths can be passed to the `gin_search_paths`
flag, and these paths should contain all Gin files used or included in your
experiment.


After fine-tuning has completed, you can parse metrics into CSV format using the
following script:

```sh
MODEL_DIR= # from Step 4 if running locally, from XManager Artifacts otherwise
VAL_DIR="$MODEL_DIR/inference_eval"
python -m t5.scripts.parse_tb \
  --summary_dir="$VAL_DIR" \
  --seqio_summaries \
  --out_file="$VAL_DIR/results.csv" \
  --alsologtostderr
```

### Metric Explanations

By default, t5x logs many metrics to TensorBoard, many of these seem similar but
have important distinctions.

The first two graphs you will see are the `accuracy` and `cross_ent_loss`
graphs. These are the *token-level teacher-forced* accuracy and cross entropy
loss respectively. Each of these graphs can have multiple curves on them. The
first curve is the `train` curve. This is calculated as a running sum than is
then normalized over the whole training set. The second class of curves have the
form `training_eval/${task_name}`. These curves are created by running a subset
(controlled by the `eval_steps` parameter of the main train function) of the
validation split of `${task_name}` through the model and calculating these
metrics using teacher-forcing. These graphs can commonly be used to find
"failure to learn" cases and as a warning sign of overfitting, but these are
often not the final metrics one would report on.

The second set of graphs are the ones under the collapsible `eval` section in
TensorBoard. These graphs are created based on the `metric_fns` defined in the
SeqIO task. The curves on these graphs have the form
`inference_eval/${task_name}`. Values are calculated by running the whole
validation split through the model in inference mode, commonly auto-regressive
decoding or output scoring. Most likely these are the metrics that will be
reported.

More information about the configuration of the datasets used for these
different metrics can be found
[here](#train-train-eval-and-infer-eval).

In summary, the metric you actually care about most likely lives under the
`eval` tab rather, than in the `accuracy` graph.

## Next Steps

Now that you have successfully fine-tuned a pre-trained model on GLUE, here are
some topics you might want to explore next:

+   [Evaluating a fine-tuned model.](eval)
+   [Running inference on a fine-tuned model.](infer)
+   [Training a model from scratch.](pretrain)

We also touch upon a few advanced topics related to fine-tuning below that might
be useful, especially when customizing your fine-tuning job.

## Advanced Topics

### `train`, `train_eval` and `infer_eval` {.no-toc}

A
[`DatasetConfig`](https://github.com/google-research/t5x/tree/main/t5x/utils.py?l=113&rcl=375475889)
object is used to configure loading SeqIO Tasks/Mixtures for training and eval.
If you take a closer look at
[`runs/finetune.gin`](https://github.com/google-research/t5x/tree/main/t5x/configs/runs/finetune.gin),
you will see that there are three `DatasetConfig` objects defined and passed to
the train function: `train_dataset_cfg`, `train_eval_dataset_cfg`,
`infer_eval_dataset_cfg`. Here's a brief description of these configs:

+   `train`: This configures the Task/Mixture that the model will be fine-tuned
    on.
+   `train_eval`: This configures the Task/Mixture that is used to compute
    training metrics on the eval split, e.g. perplexity. These metrics are
    defined in the
    [`Model`](https://github.com/google-research/t5x/tree/main/t5x/models.py;l=257-267;rcl=394045248)
    class and the eval fn is located
    [here](https://github.com/google-research/t5x/tree/main/t5x/trainer.py;l=257;rcl=398487394).
+   `infer_eval`: This configures the Task/Mixture that is used to compute
    metrics on inferred model outputs (e.g., comparing decoded model outputs and
    targets). These metrics are defined in the SeqIO Task/Mixture and the eval
    fn is located
    [here](https://github.com/google/seqio/tree/main/seqio/evaluation.py?l=423&rcl=373643592)

### Using separate SeqIO Tasks/Mixtures for fine-tuning and eval {.no-toc}

Commonly, the same SeqIO Task/Mixture is used for training and eval. It is set
by the `MIXTURE_OR_TASK_NAME` macro in your fine-tune Gin file from Step 3
above, and is passed to `train_dataset_cfg`, `train_eval_dataset_cfg`,
`infer_eval_dataset_cfg`. The `train` split is used for training and the
`validation` split is used for evals. However, you can override these params in
your fine-tune Gin config. For example, if you want to fine-tune on all GLUE
tasks but evaluate only on GLUE STS benchmark, you can override the SeqIO
Task/Mixture used for `infer_eval` in your fine-tune Gin file as follows:

```gin
include 'runs/finetune.gin'
include 'models/t5_small.gin'

MIXTURE_OR_TASK_NAME = 'glue_v002_proportional'
MIXTURE_OR_TASK_MODULE = 't5.data.mixtures'
TASK_FEATURE_LENGTHS =  {'inputs': 512, 'targets': 84}
TRAIN_STEPS = 1_500_000  # includes 1_000_000 pretrain steps
INITIAL_CHECKPOINT_PATH = 'gs://t5-data/pretrained_models/t5x/t5_small/checkpoint_1000000'
infer_eval/utils.DatasetConfig.mixture_or_task_name = 'glue_stsb_v002'
```

Other params in `finetune.gin` can be overridden in the same way.


### Defining a custom SeqIO Task/Mixture to fine-tune on {.no-toc}

Refer to [SeqIO documentation](https://github.com/google/seqio/blob/main/README.md).
