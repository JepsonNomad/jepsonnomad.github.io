---
layout: post
title: "Machine learning method for reading embedded temperature data"
author: "Christian John"
date: "4/20/2020"
output: 
  html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Background

I, like many ecologists, use remote cameras in the field. Many folks use [camera traps](https://emammal.si.edu/) to monitor wildlife populations, and others use [digital repeat photography](https://phenocam.sr.unh.edu/webcam/) (time-lapse) to focus on seasonal changes in the environment. In practice, remote photography often yields datasets of thousands or even millions of images. You can probably imagine how quickly the workload of photo management and analysis quickly piles up. 

I study [plant phenology](https://budburst.org/phenology-defined), which means I'm frequently fighting with time-lapse permitting, setup, and analysis. Some camera traps and outdoor time-lapse camera models can embed information in an infobar on each photo. For example in this [Reconyx image](https://media.springernature.com/full/springer-static/image/art%3A10.1038%2Fs41598-018-34936-0/MediaObjects/41598_2018_34936_Fig1_HTML.png?as=webp> (from [Loock et al 2018]<https://www.nature.com/articles/s41598-018-34936-0)), we can see a date-time stamp, photo sequence index, moon phase, and temperature in the upper imprinted bar.

![TL camera](https://github.com/JepsonNomad/tempKeras/blob/master/imgs/IMG_2806.jpg)

## The problem

The time-lapse camera model I use in the field can also imprint the temperature on the photo. This is super convenient when studying plant and snow phenology, because these things tend to covary with temperature (among other things). Unfortunately, while the temperature is imprinted in the photograph, that data is not stored in the image's .exif metadata. So, while it's simple to programatically access things like the timestamp or exposure settings of an image, it is not so easy to access temperature information. I decided to put together a convolutional neural network to read the temperature from my time-lapse images. The goal was to be able to input a series of images and produce a table of filenames with associated temperature measurements.

![Sample TL image](https://github.com/JepsonNomad/tempKeras/blob/master/imgs/sampleTLpic.JPG)

## The approach

To go from images to digit recognition and identification, a few things need to happen:

1. Localization - point the computer to where the numbers are. A quick-and-dirty solution to this is to crop every image to the region you're interested in.
2. Classification - identify the most likely values of the sequence of digits occupying the region defined above. This will require training and testing a CNN.

These steps are simplified in this application over other computer vision tasks, because the location where the temperature data is imprinted is consistent across every image. So, there's no need for a computer to search an image for numbers. By defining the bounding box for where temperature "lives" in the image, I crop away all of the "uninteresting" parts of the picture (uninteresting for this application - I'd much rather look at pictures of plants than temperature values; hence, training a computer to do it for me). For the digit classification step, we know there are only certain values that can occupy each position in the temperature reading: the numbers 0-9, ' ' (indicating blank space), '°', 'C', and '-' (negative temperatures are frequent in moutains). Each of these is converted into a numeric class for training, testing, and generating model predictions. I used the values 0-9 to correspond with '0'-'9', 10 for ' ', 11 for '°', 12 for 'C', and 13 for '-'. Certain conventions exist around the order of these classes (for example, '-' is always first when present, and '°' is always followed by 'C'), further facilitating the training of the network.

## The application

### Generating training data
I used a python script to crop a year's worth of imagery from one camera to just the region occupied by the temperature stamp. The CNN will ultimately be dealing with predicting 5 separate values in a sequence. Each cropped image was given a unique filename and stored in one common subdirectory for training imagery. Then I created a spreadsheet that had a column for filename, and one column for each of the 5 positions that would be evaluated. Digit columns were filled according to whatever value was in that position of that image.

![](https://github.com/JepsonNomad/tempKeras/blob/master/imgs/Train00352.JPG)

![](https://github.com/JepsonNomad/tempKeras/blob/master/imgs/WSCT0014.JPG)

### Training the CNN
A keras convolutional neural network was trained by stacking two conv2D layers, followed by a flattening step, a large dense layer, and then 5 positional dense layers that correspond with each of the 5 positions for which digits will be analyzed. I included several dropout layers to avoid overfitting, but given the consistency of the temperature stamp in imagery, I'm not sure if those were necessary. I also trained the model for far longer than necessary (100 epochs); you can see from the accuracy plots that it performed exquisitely on the validation data after only a few epochs. The core of the CNN structure was inspired by Sajal Sharma's work [here](https://sajalsharma.com/portfolio/digit_sequence_recognition).

![CNN Accuracy](https://github.com/JepsonNomad/tempKeras/blob/master/imgs/conv2D_accuracy.jpeg)
![CNN Loss](https://github.com/JepsonNomad/tempKeras/blob/master/imgs/conv2D_loss.jpeg)

### In practice
It seems like the model works quite well. Using an additional testing dataset (imagery that came from the same camera model but a different set of images from what was used for training and validation), predictions appear to be spot-on. So, I feel pretty safe about using this for the rest of my million+ images that need to be analyzed. 

## Comments

### Summary

The model I describe above shows the value of machine learning in computer vision for specific, repeatable tasks. It is also a reminder of how quickly a machine can learn how to do a simple task on its own - training the model took less than 5 minutes. With a fairly small dataset (less than 400 images) the model achieved perfect accuracy for new (but comparable) images with embedded temperature data. Speaking of which, how did the model perform with that example image I posted above? It predicted... 20°C. Hopefully it can be useful as a tool for others, or at least serve as a framework off of which others can build. All of the training data and code for this project can be found [here](***Insert website address for github).

I had originally planned to generate this model using the [Street View House Numbers dataset (SVHN)](http://ufldl.stanford.edu/housenumbers/). For individual digits I had gotten a model to achieve >90% accuracy for individual digit identification. However, a 10% error rate was far to large for my needs with a data extration tool. Furthermore, I'd been using the Format 2 imagery, which contains many extraneous numbers in images classified for 1 digit apiece. 

### Limitations

The training data for this model is derived entirely from temperature stamps stored as °C on High resolution photos produced by Wingscapes TimelapseCams. That means it's going to have strong internal validity but weak external validity. Results for any imagery that doesn't follow these parameters should be interpreted with caution. If new photos come from a different model camera, the font size, shape, etc used for temperature stamps could be different. Downscaling the imagery from different resolution base photos could have an effect on the resampled temperature imprint.

If the temperature is stored in °F I'm not sure if this model will work, because it hasn't been trained to deal with the value "F". However, a potential workaround is to identify the position in each output that is classified as a "°", and remove whatever the predicted value is after that. It should probably be tested and checked before you assume it's going to work though.







