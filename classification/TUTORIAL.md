



# Tutorial for training a classifier
This tutorial walks through all steps required to train an animal species classifier from an camera trap dataset. Instead of whole-image classification, we will focus on classification of the detected animals with unknown species.

## Requirements

### Hardware and software
We will assume that you have a Linux-based (virtual) machine running. It is also highly recommended to use a GPU for accelerating the animal detection and classifier training. The following steps will assume, that you have already set up the NVIDIA drivers and CUDA. 

Our code was tested with Python 3.6 and uses the libraries [Tensorflow](https://www.tensorflow.org/), [pycocotools](https://github.com/cocodataset/cocoapi/tree/master/PythonAPI), and the [Tensorflow object detection library](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/installation.md). 

### Data
The input to our scripts is a dataset with image-level species labels. Please refer to [http://lila.science/faq](http://lila.science/faq) for more details. The scripts will assume that the location of each image is available in the json metadata. 

This tutorial will use the [Snapshot Serengeti](http://lila.science/datasets/snapshot-serengeti) dataset as an example. 

## Copying data
First, we will copy the image data from the blob storage to the local machine. All images and the metadata will be placed in the directory `/data/serengeti`. You can run

    DATASET_DIR=/data/serengeti

to assign this path to a variable called `DATASET_DIR`. 

Let's first create that directory that directory and change to it using

    mkdir $DATASET_DIR && cd $DATASET_DIR

You can copy the image data from blob storage to your local machine, unzip the archives, and delete the zip files using

    BASEURL=https://lilablobssc.blob.core.windows.net/snapshotserengeti
    for FILE in S1 S2 S3 S4 S5 S6; 
         do azcopy --source $BASEURL/${FILE}_release.zip --destination ${FILE}_release.zip; 
         unzip -q ${FILE}_release.zip;
    done

As this process might take a while, we highly recommend running this command in a tmux or screen session. 

The same steps are required for the metadata. 

    azcopy --source $BASEURL/SnapshotSerengeti.json.zip --destination SnapshotSerengeti.json.zip
    unzip -q SnapshotSerengeti.json.zip

Now we are ready to run the code.

## Running the detector
The next step is creating a dataset suitable for classification training. 

First, clone this repository ([CameraTraps](https://github.com/Microsoft/CameraTraps)) to a local directory by running

    CAMERATRAPS_DIR=/data/CameraTraps
    git clone https://github.com/Microsoft/CameraTraps.git $CAMERATRAPS_DIR

and change to the folder with the relevant scripts

    cd $CAMERATRAPS_DIR/data_management/databases/classification

We will now use `make_classification_dataset.py` to generate the dataset. The script will create two output directories, one for a COCO-style classification dataset and one for a tfrecords output. While we actually only need the tfrecords output, it is recommended to create a COCO-style dataset as well. It is necessary for analyzing the created data and also contains the detected bounding boxes for each image. We will define the location of both output folders by running

    COCO_STYLE_OUTPUT=/data/serengeti_cropped_coco_style
    TFRECORDS_OUTPUT=/data/serengeti_cropped_tfrecords


We also need a frozen detection model as generated by the Tensorflow object detection API. We will assign the path to this frozen graph to a variable:

    FROZEN_DETECTION_GRAPH=/tmp/frozen_inference_graph.pb

Now the detection can be started by running 

    python make_classification_dataset.py \
    $DATASET_DIR/SnapshotSerengeti.json \
    $DATASET_DIR/ \
    $FROZEN_DETECTION_GRAPH \
    --coco_style_output $COCO_STYLE_OUTPUT \
    --tfrecords_output TFRECORDS_OUTPUT \
    --location_key location \
    --exclude_categories human empty

In addition to detection and cropping of detected animals, the script will also divide the data into training and testing splits based on the locations. If you would like to use a different field name for splitting the data, you can use the `--location_key` flag. 

The script will run for days or weeks depending on the dataset size. More details on the flags are discussed in the [classification README](https://github.com/Microsoft/CameraTraps/tree/master/classification#animal-detection-and-cropping). You can go to the folder `$COCO_STYLE_OUTPUT` and analyze the generated images. The script generates one subfolder for each category. 

## Dataset statistics
The number of generated images depends on the number of detected animals in the dataset as well as the how many of the images have only labeled with one species. We need the exact number of images for the classifier training. It can be computed using the script `cropped_camera_trap_dataset_statistics.py` in the same folder.

    python cropped_camera_trap_dataset_statistics.py \
    $DATASET_DIR/SnapshotSerengeti.json \
    $COCO_STYLE_OUTPUT/train.json \
    $COCO_STYLE_OUTPUT/test.json \
    --classlist_output $COCO_STYLE_OUTPUT/classlist.txt

The script will print a complete overview of the generated data, which looks like this:

    loading annotations into memory...
    Done (t=23.26s)
    creating index...
    index created!
    Statistics of the training split:
    Locations used:
    ['B04', 'B05', 'B06', ...
    In total  49  classes and  1730648  images.
    Classes with one or more images:  47
	Images per class:
	ID    Name            Image count
        0 empty                     0
        1 human                     0
        2 gazelleGrants          7968
        3 reedbuck               2332
    ...
    
    Statistics of the testing split:
    Locations used:
	['C05', 'C08', ...
    In total  49  classes and  439292  images.
	Classes with one or more images:  46
	Images per class:
	ID    Name            Image count
	    0 empty                     0
	    1 human                     0
	    2 gazelleGrants          1379
	    3 reedbuck                830
	...
	
This tells us that there are in total 49 classes. The training split has 1730648 images and the testing split 439292. 

The output also contains the class distribution for each split, which allows for generating nice figures using a spreadsheet program of your choice. Copy all the lines

		0 empty                     0
	    1 human                     0
	    2 gazelleGrants          1379
	    ....
 
with image counts to a text file, replace all consecutive spaces by a single comma using regular expressions, and open the file as CSV in a spreadsheet program. The program should detect the class names and image counts as separate columns. You can now create a plot such as:

![Serengeti class distribution](tutorial_images/serengeti_class_distribution.png?raw=true "Serengeti class distribution")



## Training a classifier
With this information, we can move on to the classifier training. The training requires three things:

1. The TFRecords in `$TFRECORDS_OUTPUT` as generated above
2. A pre-trained [Inception V4 model](http://download.tensorflow.org/models/inception_v4_2016_09_09.tar.gz)
3. Slight modifications to the Tensorflow slim code in `$CAMERATRAPS_DIR/classification/tf-slim`

We already have the TFRecords, so we can directly jump to step 2, downloading the model. Let's put it in a separate directory:

    PRETRAINED_DIR=/data/pretrained
    mkdir $PRETRAINED_DIR && cd $PRETRAINED_DIR
    wget -O inc4.tar.gz http://download.tensorflow.org/models/inception_v4_2016_09_09.tar.gz
    tar xzf inc4.tar.gz 

Now, we continue with the code modifications. Each dataset requires a separate dataset description file, which is located in 
`$CAMERATRAPS_DIR/classification/tf-slim/datasets/`. We will create a file for our Serengeti dataset from scratch to explain the procedure:

    cd $CAMERATRAPS_DIR/classification/tf-slim/datasets/
    cp cct.py serengeti.py

Open the file and adjust lines 20 and 22 to match the our dataset numbers:

    SPLITS_TO_SIZES = {'train': 1730648, 'test': 439292}
    _NUM_CLASSES = 49

Close and save the file. This description now gets connected to the rest of the code by the file `dataset_factory.py`. Open it with

    vim $CAMERATRAPS_DIR/classification/tf-slim/datasets/dataset_factory.py

and add an import as well as an entry to the list of datasets at line 37. The list of datasets is a dictionary called `datasets_map` and should now look like this:

    ...
    from datasets import serengeti  # <-- This is new
    datasets_map = {
        'cifar10': cifar10,
        'flowers': flowers,
        'imagenet': imagenet,
        'mnist': mnist,
        'cct': cct,
        'wellington': wellington,
        'serengeti': serengeti, # <-- This is new
        'nacti': nacti
    }
    ...

Now everything is prepared. The easiest way to start the training is by adapting one of the existing training scripts. We first switch to the classification directory.

    cd $CAMERATRAPS_DIR/classification

This folder contains several sample scripts for the training. As before, we will adapt an existing file to our needs:

    cp train_cct_inception_v4.sh train_serengeti_inception_v4.sh

Open `train_serengeti_inception_v4.sh` with a text editor and adjust
- `DATASET_NAME` to the key used in the dict `datasets_map` above, in our case `serengeti`
- `DATASET_DIR` to the path to your TFRecords folder `$TFRECORDS_OUTPUT`
- `CHECKPOINT_PATH` to the path of the checkpoint located at `$PRETRAINED_DIR/inception_v4.ckpt`

In our case, these three values would look like this

    DATASET_DIR=/data/serengeti_cropped_tfrecords
    DATASET_NAME=serengeti
    CHECKPOINT_PATH=/data/pretrained/inception_v4.ckpt

You might also want to adjust the number of training steps by changing the value of `--max_number_of_steps` at [line 51](https://github.com/Microsoft/CameraTraps/blob/classification/classification/train_serengeti_inception_v4.sh#L51). A good starting value is 30 epochs, which translates to 

    NUM_EPOCHS * NUM_TRAINING_IMAGES / BATCH_SIZE

steps. The longer you run the training, the better might be the accuracy. 

We are now ready to start the training! Go to the Tensorflow slim directory with 

    cd $CAMERATRAPS_DIR/classification/tf-slim

and start the training with 

    bash ../train_serengeti_inception_v4.sh

The training will take several days to complete. It is a two-step training, in which we first train only the last layer of the network. Afterward, all parameters are trained jointly. 

## Evaluating the classifier
The sample training scripts will run an evaluation run on the test data after the training finished. This evaluation run can be repeated later as well. We will adapt an existing evaluation script for this purpose. First, switch to the directory containing the sample scripts by executing

    cd $CAMERATRAPS_DIR/classification

and copy one of the existing scripts with

    cp eval_cct_inception_v4.sh eval_serengeti_inception_v4.sh

This script needs similar adjustments as the training script created in the previous section. Open the script `eval_serengeti_inception_v4.sh` with a text editor like `vim` by executing

    vim eval_serengeti_inception_v4.sh

and set the variables in the top:
- `DATASET_NAME` to the key used in the training script, in our case `serengeti`
- `DATASET_DIR` to the path to your TFRecords folder `$TFRECORDS_OUTPUT`
- `CHECKPOINT_PATH` to the subfolder `all` of your log directory

One log folder is created for each training run in the directory 

    $CAMERATRAPS_DIR/classification/tf-slim/log

and the folder name contains the starting date and time. Double check that that you pick the correct path as the evaluation will run even with an invalid path. In our case, the top of the script looks like this

    DATASET_DIR=/data/serengeti_cropped_tfrecords
    DATASET_NAME=serengeti
    CHECKPOINT_DIR=/data/CameraTraps/classification/tf-slim/log/2019-02-22_07.05.54_well_incv4/all/

The script is now ready for execution. Change to the `tf-slim` folder by executing

    cd $CAMERATRAPS_DIR/classification/tf-slim

and start the evaluation run with 

    bash ../eval_serengeti_inception_v4.sh

As before, the accuracy will be printed at the end:

    ...
    INFO:tensorflow:Evaluation [3951/4393]
    INFO:tensorflow:Evaluation [4390/4393]
    INFO:tensorflow:Evaluation [4393/4393]
    eval/Accuracy[0.863694489]eval/Recall_5[0.969760954]

    INFO:tensorflow:Finished evaluation at 2019-05-07-03:34:21

The outputs `eval/Accuracy` and `eval/Recall_5` correspond to the top-1 and top-5 accuracy with values in [0,1]. 

## Single-image prediction
The trained model can be deployed and used for single-image prediction. The first step is exporting the model definition and creating a frozen graph. Start by changing to the `tf-slim` folder 

    cd $CAMERATRAPS_DIR/classification/tf-slim

and by making sure, that your `CHECKPOINT_PATH` points to the subfolder `all` of your training log directory. In our case it looks like this:

    CHECKPOINT_DIR=/data/CameraTraps/classification/tf-slim/log/2019-02-22_07.05.54_well_incv4/all/

We can now start exporting the model definition by running
	
	python ../export_inference_graph_definition.py \
	    --model_name=inception_v4 \
	    --output_file=${CHECKPOINT_DIR}/inception_v4_inf_graph_def.pbtxt \
	    --dataset_name=serengeti \
	    --write_text_graphdef=True

where `$CHECKPOINT_DIR` is the path to the log directory of the model training as above. The export command will create a file called `inception_v4_inf_graph_def.pbtxt` in the log directory, which represents the structure of the classification model. The file does not contain the learned model parameters yet. These values are stored in the checkpoint files starting with `model.ckpt...`. 

We will now fuse the model structure and learned parameter values into one file, which is called frozen graph:

	python ../freeze_graph.py \
	    --input_graph=${CHECKPOINT_DIR}/inception_v4_inf_graph_def.pbtxt \
	    --input_checkpoint=`ls ${CHECKPOINT_DIR}/model.ckpt*meta | tail -n 1 | rev | cut -c 6- | rev` \
	    --output_graph=${CHECKPOINT_DIR}/frozen_inference_graph.pb \
	    --input_node_names=input \
	    --output_node_names=output \
	    --clear_devices=True

You should be able to run this command without any modification as the only bash variable here is `$CHECKPOINT_DIR`. The script will create a file called `frozen_inference_graph.pb` in the folder `$CHECKPOINT_DIR`, which is a deployable self-contained inference model. This model also contains all image preprocessing steps required.

We now can predict the category of a new image as follows. First, change to the following directory

    cd $CAMERATRAPS_DIR/classification

This folder contains the script `predict_image.py`, which takes the frozen graph, a list of class names, and an image as input. It will then read the image, pass it to the classification model, and print the five most likely categories. 

	python predict_image.py \
	--frozen_graph ${CHECKPOINT_DIR}/frozen_inference_graph.pb \
	--class_list $COCO_STYLE_OUTPUT/classlist.txt \
	--image_path $COCO_STYLE_OUTPUT/gazelleThomsons/S1/R10/R10_R1/S1_R10_R1_PICT1199_1.JPG

This command will analyze the test image `$COCO_STYLE_OUTPUT/gazelleThomsons/S1/R10/R10_R1/S1_R10_R1_PICT1199_1.JPG`, which looks like

![Gazelle](tutorial_images/S1_R10_R1_PICT1199_1.JPG?raw=true "Gazelle")

and print the following output:

	...
	Prediction finished. Most likely classes:
	    "gazelleThomsons" with confidence 90.73%
	    "gazelleGrants" with confidence 3.97%
	    "zebra" with confidence 1.96%
	    "warthog" with confidence 0.41%
	    "otherBird" with confidence 0.38%


