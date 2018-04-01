# emotion-detection-raspberry

In this repository we will first apply transfer learning to import the tensorflow Inception model and we will retrain the last layer using the CK+ dataset that we will expand with pictures scraped from Google Image.

The achieved model will then be implemented on a Raspberry Pi 3 B+

Further improvements will consist in:
- logging detections in a MySQL db and expose these logs in a RESTful API.
- Using the KDEF dataset in addition to the CK+ dataset for better accuracy.

## Pre-requisite

Having a Raspberry Pi 3 B+

Install Tensorflow on it

`sudo apt-get install python3-pip python3-dev`

```
wget https://github.com/samjabrahams/tensorflow-on-raspberry-pi/releases/download/v1.1.0/tensorflow-1.1.0-cp34-cp34m-linux_armv7l.whl
sudo pip3 install tensorflow-1.1.0-cp34-cp34m-linux_armv7l.whl
```

If the above command doesn't work try renaming the wheel file by replacing 34 by 35 (your python version).
Finally if you have mock installed, uninstall then reinstall:

```
sudo pip3 uninstall mock
sudo pip3 install mock
```

Install opencv:
`sudo pip3 install opencv-python`

## Transfer learning

Inception is a TF pre trained model. The training has been done on millions of images classifiying them in 1000 classes. Training such a big model would be time consuming and expensive. That's where transfer learning becomes useful.
The first layers of Inception are basically edges and shape recognition layers. This is a task which will not change whatever we want to recognize. Therefore we can save a lot of training by reusing the weights of these layers and only training our last layers to classify properly what we want to figure out: emotions on faces.

In this retraining we will feed the CK+ dataset to the Inception model the following way:
- clone tensorflow master repository locally
- in tf local repo root folder create a new directory. Mine is called retrained_data.
- copy the CK+ folders (angry, contempt...) in retrained_data/dataset/ and convert pictures in JPG if they are not already in this format
- retrain the model using the following command:

```
python ./tensorflow/examples/image_retraining/retrain.py \
--bottleneck_dir=/retrained_data/bottlenecks \
--how_many_training_steps 500 \
--model_dir=/retrained_data/inception \
--output_graph=/retrained_data/retrained_graph.pb \
--output_labels=/retrained_data/retrained_labels.txt \
--image_dir /retrained_data/dataset
```

Now that you have a trained model, connect by sftp to your raspberry and copy the files retrained_graph.pb, retrained_labels.txt from your retrained_data folder into a new folder on the raspberry.

# PiCamera

We are using the official raspberry pi camera v2.1. We need the `picamera` module for python.

A simple test to see if it works:

```import picamera 
import time

with picamera.PiCamera() as camera:
    camera.resolution = (2592, 1944)
    camera.start_preview()
    time.sleep(2)
    camera.exif_tags['IFD0.Copyright'] = 'Copyright (c) 2018'
    camera.capture('./foo.jpg')
    camera.stop_preview()
```
This captures a screenshot 2s after activation of the camera then close the video feed preview.

## OpenCV

We use the HaarCascade Classifier to find out faces in the video stream.
On this first version we loop until 20 faces are detected. Each bounding box is saved as an image file and then fed to the TF model.