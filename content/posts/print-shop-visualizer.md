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

## Step 3: Handle http connections

Go once again has first class support for http and http servers built in. So first we must import `net/http` and a router. Im using mux here: `github.com/gorilla/mux`

```go
import (
	"log"
	"net/http"
	"github.com/gorilla/mux"
	"os"
)
```

The next thing to do is setup the router and http server. This can be done in just 3 lines! This will be the entrypoint to our program.

```go
func main() {
  // Create router
	router := mux.NewRouter().StrictSlash(true)
  // Only accept POST requests
	router.Methods("POST").Path("/").HandlerFunc(handleImage)
  // Start the server and if it exits log the error
	log.Fatal(http.ListenAndServe(":8080", router))
}
```

That was suprisingly easy wasnt it. But what about that handleImage function. That brings us to the next part which is processing the request.

## Step 4: Process incoming requests

The first step is to create a function that gets called by the router whenever the specified path is requested. This takes a `ResponseWriter` to write back to the client and a `Request` with info on the request made.

```go
func handleImage(w http.ResponseWriter, r *http.Request) {

}
```

First we must process the incoming form data. This can be done in one line:

```go
r.ParseMultipartForm(32 << 20)
```

This specifies a maxium file size of 32MiB and parses the other fields as well. We then want to access said file.

```go
file, _, err := r.FormFile("uploadfile")
if err != nil {
  fmt.Println(err)
  w.WriteHeader(http.StatusBadRequest)
  return
}
defer file.Close()
```

This opens the file just as if it was opened from disk. If there was an error (example: the field doesn't exist) then we send back a 400 Bad Request error and exit the function. The `defer file.Close()` makes sure the file is closed at the end of the function even if we error.

We now need to know what base to use. We can get the form field to do so. If the field was not specified we get a blank string so we return a 400 error again.

```go
baseName := r.FormValue("base")
if baseName == "" {
  w.WriteHeader(http.StatusBadRequest)
  return
}
```

We finaly have everything needed to process the image. So we call our previously defined `loadImageConfig` function to get everyting ready for processing. If the config can't be found we return a 404 error otherwise something unknown went wrong and we return a 500 error.

```go
base, err := loadImageConfig(baseName)
if err != nil {
  if _, ok := err.(*os.PathError); ok {
    w.WriteHeader(http.StatusNotFound)
    return
  } else {
    w.WriteHeader(http.StatusInternalServerError)
    return
  }
}
```

The last thing to do is process the image and return it. I'll get on to how we process the image next but first I'll finish this function and return the processed image. We pass the base, the overlay image and the writer so that the function can write back the image. If there was an error we return a 500 error.

```go
err = processImage(base, file, w)
if err != nil {
  w.WriteHeader(http.StatusInternalServerError)
}
```

All of this together makes this:

```go
import (
	"log"
	"net/http"
	"github.com/gorilla/mux"
	"os"
)

func check(e error) {
	if e != nil {
		panic(e)
	}
}

func handleImage(w http.ResponseWriter, r *http.Request) {
	r.ParseMultipartForm(32 << 20)
	file, _, err := r.FormFile("uploadfile")
	if err != nil {
		fmt.Println(err)
		w.WriteHeader(http.StatusBadRequest)
		return
	}
	defer file.Close()

	baseName := r.FormValue("base")
	if baseName == "" {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	base, err := loadImageConfig(baseName)
	if err != nil {
		if _, ok := err.(*os.PathError); ok {
			w.WriteHeader(http.StatusNotFound)
			return
		} else {
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
	}

	err = processImage(base, file, w)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
	}
}

func main() {
	router := mux.NewRouter().StrictSlash(true)
	router.Methods("POST").Path("/").HandlerFunc(handleImage)
	log.Fatal(http.ListenAndServe(":8080", router))
}
```

## Step 5: Processing the image

So that function we called in previous step `processImage` needs writing. Let's do that! Go has first class support for image editing with `image/draw` (Go seems to have first class support for nearly everything right?). So let's import eveything we'll need. I'm using `github.com/disintegration/imaging` for some helper functions. I'm importing `image/png` without `_` because I'll use it to export the final image.

```go
import (
	"image"
	"image/draw"
	"image/png"
	_ "image/jpeg"
	"github.com/disintegration/imaging"
	"io"
)
```

So let's start with a blank function that takes the base, overlay and output writer in: 

```go
func processImage(base baseImage, source io.Reader, out io.Writer) error {

}
```

First we need to setup some things for later. Let's get the size of the final image a create a blank image to draw everything over.

```go
size := base.backImage.Bounds()
finalImage := image.NewNRGBA(size)
```

Then we'll try to decode the image from the client. If it fails we exit and return the error.

```go
sourceImage, _, err := image.Decode(source)
if err != nil {
  return err
}
```

We then calculate the width and hight of the area we can draw on and fit the source image withing that rectangle. I use a image.Pt to store width and height here not X,Y values but it works.

```go
boundsSize := image.Pt(base.bottomRightBound.X-base.topLeftBound.X,
		base.bottomRightBound.Y-base.topLeftBound.Y)

sourceImage = imaging.Fit(sourceImage, boundsSize.X, boundsSize.Y, imaging.Lanczos)
sourceImageSize := sourceImage.Bounds()
```

This bit is somewhat complicated. We need to calculate where to palce the overlay. I won't explain bacause I don't even know what I wrote but it works. We also then calulate bounds for the placement from the position and the size.

```go
pos := image.Pt((base.topLeftBound.X+boundsSize.X/2)-(sourceImageSize.Max.X/2),
		(base.topLeftBound.Y+boundsSize.Y/2)-(sourceImageSize.Max.Y/2))
bounds := image.Rect(pos.X, pos.Y, pos.X+boundsSize.X, pos.Y+boundsSize.Y)
```

Finally we draw all the images in the correct order, write it out to the client and return `nil` error.

```go
draw.Draw(finalImage, finalImage.Bounds(), base.backImage, image.Pt(0, 0), draw.Over)
draw.Draw(finalImage, bounds, sourceImage, image.Pt(0, 0), draw.Over)
draw.Draw(finalImage, finalImage.Bounds(), base.frontImage, image.Pt(0, 0), draw.Over)

png.Encode(out, finalImage)
return nil
```

All of that together looks like this:

```go
import (
	"image"
	"image/draw"
	"image/png"
	_ "image/jpeg"
	"github.com/disintegration/imaging"
	"io"
)

func processImage(base baseImage, source io.Reader, out io.Writer) error {

	size := base.backImage.Bounds()

	finalImage := image.NewNRGBA(size)

	sourceImage, _, err := image.Decode(source)
	if err != nil {
		return err
	}

	boundsSize := image.Pt(base.bottomRightBound.X-base.topLeftBound.X,
		base.bottomRightBound.Y-base.topLeftBound.Y)

	sourceImage = imaging.Fit(sourceImage, boundsSize.X, boundsSize.Y, imaging.Lanczos)
	sourceImageSize := sourceImage.Bounds()

	pos := image.Pt((base.topLeftBound.X+boundsSize.X/2)-(sourceImageSize.Max.X/2),
		(base.topLeftBound.Y+boundsSize.Y/2)-(sourceImageSize.Max.Y/2))
	bounds := image.Rect(pos.X, pos.Y, pos.X+boundsSize.X, pos.Y+boundsSize.Y)

	draw.Draw(finalImage, finalImage.Bounds(), base.backImage, image.Pt(0, 0), draw.Over)
	draw.Draw(finalImage, bounds, sourceImage, image.Pt(0, 0), draw.Over)
	draw.Draw(finalImage, finalImage.Bounds(), base.frontImage, image.Pt(0, 0), draw.Over)

	png.Encode(out, finalImage)
	return nil
}
```

## All together now

I decided to split this into 3 files. Go is really nice in that any files in the same package (`main` in this case) cant just see each other without imports. The 3 files are:

- main.go - Entry-point and HTTP server
- load.go - Functions for loading images and config
- process.go - Function that actually does the rendering

Now thats all good but how do we deploy this. The answer as always is Docker! Because this is a go file and we don't want a 500MB image with all possible standard libraries we'll have to statically link the executable. This is done by setting the `CGO_ENABLED` variable to 0. So our compile command would be (in the project root).

```bash
CGO_ENABLED=0 go build
```

This will take a while as it has to compile all the libraries we used. When it's done we should have one `printshop` file that can run anywhere. So now we need a docker container to run it all in. Our `Dockerfile` shall be:

```dockerfile
FROM scratch

COPY printshop /
COPY config/ /config
COPY imgs/ /imgs

CMD ["/printshop"]
```

The `FROM scratch` starts us off with absolutely nothing in our container. We don't need any libraries so this is perfect and makes the smallest of images. We also need to copy in our config and images for the server but these could be mounted on a persistent form of storage to allow easier updates, or updates from another control container.

## That's all

I'm not going to cover deployment here but if you want a template to use for Kubernetes you can use the one in my previous post [A useful starting deployment for Kubernetes](https://misell.cymru/posts/starter-kubes-deploy/). The full source is on github [here](https://github.com/FluidMediaProductions/printshop). This is under the GPLv3 license. Normally I'd release things under the MIT license but this is for work and we use use a different license there.