---
title: "Reproducible Pitch Presentation"
author: "Kevin Tham"
date: "June 7, 2018"
framework   : io2012        # {io2012, html5slides, shower, dzslides, ...}
highlighter : highlight.js  # {highlight.js, prettify, highlight}
hitheme     : tomorrow      # 
widgets     : []            # {mathjax, quiz, bootstrap}
mode        : selfcontained # {standalone, draft}
knit        : slidify::knit2slides
---

<style>
pre {
  white-space: pre !important;
  overflow-y: scroll !important;
  height: 50vh !important;
}
</style>





## Digit Recognition App

This app takes user input in the form of a handwritten digit from 0-9, and as an output, returns the probabilities that the neural network model associates the input with a particular digit.

It is hoped that this project can be extended to an online model where the model is continuously trained on user input, and as a more ambitious extension to be able to recognise alphabets and even full sentences.

---

## Server Calculations (1)

The javascript code below reads in the input from the user. When the 'predict' button is clicked, the pixel data of the handwritten digit is transferred to the shinyServer.


```js
var $range = $(".js-range-slider");

var canvas = new fabric.Canvas("canvas");
canvas.isDrawingMode = true;

$range.ionRangeSlider({
    onChange: function(data) {
        canvas.freeDrawingBrush.width = $range.data("from");
    }
});

canvas.freeDrawingBrush.width = $range.data("from");
canvas.freeDrawingBrush.color = "#000000";
canvas.backgroundColor = "#ffffff";
canvas.renderAll();

$("#clear").click(function(){
    canvas.clear();
    canvas.backgroundColor = "#ffffff";
    canvas.renderAll();
});

$("#predict").click(function(){
    var img_array = canvas.getContext().getImageData(0, 0, 
                                     canvas.width, canvas.height).data;
    Shiny.onInputChange("img_array", img_array);
});
```

---

## Server Calculations (2)

The pixel data is then processed and fed into a neural network model to be classified. The result is then pushed to the UI via a ggplot2 bar graph.


```r
library(shiny)
library(imager)
library(ggplot2)
library(h2o)

shinyServer(function(input, output) {
  
  img_grey <- observeEvent(input$img_array, {
    
    
    # loading message
    withProgress(message='Performing Calculations:', value=1, {
      
      # rescale values to between 0-1
      img <- 255 - unlist(input$img_array)
      
      # obtain width of image
      width <- sqrt(length(img)/4)
      
      # reshape vector into array and remove transparency dimension
      img_arr <- aperm(array(img, dim=c(4, width, width)), c(2,3,1))[,,1:3]
      
      # convert format for imager manipulation to resize and greyscale 
      # and rescale between 0-1
      im <- as.cimg(img_arr)
      resized_img <- resize(im, 28, 28)
      test_vec <- as.vector(grayscale(resized_img))/255
      
      # reformat data
      test_arr <- array(test_vec, dim=c(1,784))
      test_df <- data.frame(test_arr)
      names(test_df) <- 1:784

      # To obtain predictions from trained model
      dfstr <- sapply(1:ncol(test_df), function(i) 
        paste(paste0('\"', names(test_df)[i], '\"'), test_df[1,i], sep = ':'))
      
      json <- paste0('{', paste0(dfstr, collapse = ','), '}')

      predict <- h2o.predict_json(model = 'www/nn_model.zip', json = json)
      prediction <- as.vector(predict)
      
      # Bar plot of prediction
      output$predict <- renderPlot({
        ggplot(data.frame(digit=factor(0:9), 
                          prob=as.numeric(prediction[[3]])), aes(x=digit,y=prob)) + 
          geom_bar(stat='identity') + labs(title='Model Prediction of Input Digit')
      })
    
    })
    
  })
  
})
```

---

## Application Appearance

The image below showcases the user interface of the application.

<img src="shinyapp.png" title="plot of chunk unnamed-chunk-3" alt="plot of chunk unnamed-chunk-3" width="1000px" />






