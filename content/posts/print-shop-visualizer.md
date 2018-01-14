---
date: "2018-01-14T10:43:31Z"
title: "A micro-service visualizer for a print shop in golang"
authors: []
tags:
  - go
  - micro-services
draft: true
toc: true
typora-copy-images-to: ../../static/imgs
typora-root-url: ../../static
---

First things first some background; This is a project for a client of mine at [Fluid Meida](https://fluidmedia.wales), the print shop wanted a new website and one of the main additions was a page where people could view what their prints would look like. Somewhat like what you get on teespring etc. 

## Step 0: Pre-processing of images

So the first thing is to get a base image to add prints to. I choose a hoodie because it has the drawstring which must be put on top.

![hoodie](/imgs/hoodie.png#center)

So next is to do some image editing (not my strong point) so that the drawstring is separate from the hoodie. So heres the base (luckily the pattern is simple so the heal tool in gimp work perfectly):

![hoodie back](/imgs/hoodie-b.png#center)

And here's the drawstrings. This bit I had some trouble with due to their shape but i think for a test it's acceptable. When we put this in production my friend who is great at these sort of things will edit them to a higher degree. I don't know why theres a white line on the outline but hey it's only a test.

![hoodie front](/imgs/hoodie-f.png#center)

So thats the non programming bit out of the way now we can get to actual programming. I chose to write this in [Golang](https://golang.org/) due to it's speed and the fact I've recently discovered it and like it (most of my choices are 'do I like it?').

## Step 1: Load images

Go has first class support for many things some of which I'll use in this (images, json). So to load an image is quite easy. First you import the `image` library and each type of image you want (`image/png`, `image/jpeg`) with an `_` in front so that you tell Go that you only want them for their effects and not to call them directly (Go will error on compile if you import something without using it). We also need to import `io/ioutil` and `os` to load the image data from file.

```go
import (
  "image"
  _ "image/png"
  _ "image/jpeg"
  "io/ioutil"
  "os"
)
```

Next we actually load the images. First we check the images actually exist with `os.Stat` and `os.IsNotExist` to check the error return. We `panic` if the back doesn't exist but it doesn't matter if there isn't a front because we can just not put one on the render (t-shirts for example wont have front).

```go
backFile := "imgs/hoodie-b.png"
frontFile := "imgs/hoodie-f.png"
frontExists := true

if _, err := os.Stat(backFile); os.IsNotExist(err) {
  panic("Base file does not exist")
}
if _, err := os.Stat(frontFile); os.IsNotExist(err) {
  frontExists = false
}
```

After checking for the existence of the files we can load them ad images. This is completed by calling `os.Open` on the file path then `image.Decode` on the resulting file pointer.

```go
back, err := os.Open(backFile)
defer back.Close()
check(err)
backImage, _, err := image.Decode(back)
check(err)
```

The check function here is a simple helper that panics if the error is not `nil`

```go
func check(e error) {
	if e != nil {
		panic(e)
	}
}
```

Next we can load the front image if it exists in the same way or if it doesn't create a blank image of the same size as the back. This also checks that the front and back are the same size and panics if they are not, I could have made it so that the front image could be smaller and it placed it in the middle but it's simpler just to edit it correctly in the first place.

```go
if frontExists {
  front, err := os.Open(frontFile)
  defer front.Close()
  check(err)
  frontImage, _, err = image.Decode(front)
  check(err)
  if frontImage.Bounds() != backImage.Bounds() {
    panic("Front and back images are not the same size")
  }
} else {
  frontImage = image.NewNRGBA(backImage.Bounds())
}
```

All of this can now be put in a function to load the two images and return them.

```go
func loadBaseImages(backName string, frontName string) (image.Image, image.Image) {
	backFile := "imgs/"+backName
	frontFile := "imgs/"+frontName
	frontExists := true

	if _, err := os.Stat(backFile); os.IsNotExist(err) {
		panic("Base file does not exist")
	}
	if _, err := os.Stat(frontFile); os.IsNotExist(err) {
		frontExists = false
	}

	back, err := os.Open(backFile)
	defer back.Close()
	check(err)
	backImage, _, err := image.Decode(back)
	check(err)
	var frontImage image.Image
	if frontExists {
		front, err := os.Open(frontFile)
		defer front.Close()
		check(err)
		frontImage, _, err = image.Decode(front)
		check(err)
		if frontImage.Bounds() != backImage.Bounds() {
			panic("Front and back images are not the same size")
		}
	} else {
		frontImage = image.NewNRGBA(backImage.Bounds())
	}

	return backImage, frontImage
}
```

## Step 2: Load the base image from a config file

So great we can load images, but how do we know what images to load? We can have a config directory with json files telling the program information about the images available such as the filenames and bounds in which a print can be placed. 

Our json config will look like this, with a filename of hoodie.json specifying that this is for the hoodie print.

```json
{
  "back": "hoodie-b.png",
  "front": "hoodie-f.png",
  "topLeft": {
    "X": 164,
    "Y": 107
  },
  "bottomRight": {
    "X": 387,
    "Y": 315
  }
}
```

If there was no front image then the `front` key would be empty.

Let's start by defining a way of storing all this data one loaded. A struct is perfect for this.

```go
type baseImage struct {
	backImage image.Image
	frontImage image.Image
	topLeftBound image.Point
	bottomRightBound image.Point
}
```

This stores the two images loaded from the previous step and two point for the printing bounds. Next we need a function that can load these in from file. Lucky Go has first class support for json with the `encoding/json` library. First we read in the config file and check there was no error.

```go
jsonText, err := ioutil.ReadFile("config/"+name+".json")
if err != nil {
  return baseImage{}, err
}
```

Returning the struct with no values set will make the images blank 0x0 images and the points (0,0).

We then setup a variable to store the json data and call `json.Unmarshal` to parse the json, `panic`ing if we encounter an error.

```go
var dat map[string]interface{}

if err := json.Unmarshal([]byte(jsonText), &dat); err != nil {
  panic(err)
}
```

Next we call the function we previously wrote to load the images from the paths specified in the config file.

```go
backImage, frontImage := loadBaseImages(dat["back"].(string), dat["front"].(string))
```

Because of the fact that json can store many types we defined the `dat` variable as a generic `interface` but we then need to add a `.(string)` to access the underlying string type.

We can then load in the top-left and bottom-right print bounds.

```go
topLeftBound := image.Pt(int(dat["topLeft"].(map[string]interface{})["X"].(float64)), int(dat["topLeft"].(map[string]interface{})["Y"].(float64)))
bottomRightBound := image.Pt(int(dat["bottomRight"].(map[string]interface{})["X"].(float64)), int(dat["bottomRight"].(map[string]interface{})["Y"].(float64)))
```

This looks complicated but really its just a lot of accessing underlying types. First we access the `topLeft` value which stores data like this:

```json
"topLeft": {
  "X": 164,
  "Y": 107
}
```

We then need to get to the object containing the `X` and `Y` values so we need to access the `map[string]interface{}` type again. Then we can get to the `X` value and access the underlying value of `float64`. I know that those are not floats but all json numbers could be floats so Go loads them as floats. We than encapsulate this all in `int()` to cast it to an int for the `image.Point` type. This is then repeated for the `Y` value and `bottomRight`.

This is then all put in the struct and return it with `nil` error.

```go
return baseImage{
  backImage: backImage,
  frontImage: frontImage,
  topLeftBound: topLeftBound,
  bottomRightBound: bottomRightBound,
}, nil
```

This all put together makes this function:

```go
type baseImage struct {
	backImage image.Image
	frontImage image.Image
	topLeftBound image.Point
	bottomRightBound image.Point
}

func loadImageConfig(name string) (baseImage, error) {
	jsonText, err := ioutil.ReadFile("config/"+name+".json")
	if err != nil {
		return baseImage{}, err
	}

	var dat map[string]interface{}

	if err := json.Unmarshal([]byte(jsonText), &dat); err != nil {
		panic(err)
	}

	backImage, frontImage := loadBaseImages(dat["back"].(string), dat["front"].(string))

	topLeftBound := image.Pt(int(dat["topLeft"].(map[string]interface{})["X"].(float64)),
		int(dat["topLeft"].(map[string]interface{})["Y"].(float64)))
	bottomRightBound := image.Pt(int(dat["bottomRight"].(map[string]interface{})["X"].(float64)),
		int(dat["bottomRight"].(map[string]interface{})["Y"].(float64)))

	return baseImage{
		backImage: backImage,
		frontImage: frontImage,
		topLeftBound: topLeftBound,
		bottomRightBound: bottomRightBound,
	}, nil
}
```

