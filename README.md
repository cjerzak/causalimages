# `causalimages`: `R` Package for Causal Inference with Earth Observation, Bio-medical, and Social Science Images 

*This software is under beta release. The first stable release will be sometime in spring of 2023. Email `connor.jerzak@gmail.com` for feature requests.*

# Repository Structure
This repository contains two main components. The first component is code of the `causalimages` software package. At the moment, the package includes code for performing the image-based heterogeneity decomposition described in Jerzak, Johansson, and Daoud (2023). In future releases, we will add functionality for using images in observational causal inference (i.e., as a proxy for confounding variables). 

The second component is replication data. The replication data for the Youth Opportunities Experiment is found in `data/Replication Data/YOP Experiment`. This includes both outcome data and geo-referenced satellite images. We will keep adding to this replication repository with new research being performed over time. 


# Package Download Instructions 
From within `R`, you may download via the `devtools` package. In particular, use 

```
devtools::install_github(repo = "cjerzak/causalimages-software/causalimages")
```

Then, to load the software, use 
```
library(   causalimages  ) 
```

# Tutorial
## Load in Tutorial Data
After we've loaded in the package, we can get started running an analysis. We'll start by loading in tutorial data: 
```
data(  CausalImagesTutorialData )
```
Once we've read in the data, we can explore its structure: 
```
# outcome, treatment, and covariate information: 
summary( obsW ) # treatment vector 
summary( obsY ) # outcome vector 
summary( LongLat ) # long-lat coordinates for each unit
summary( X ) # other covariates 

# image information: 
dim( FullImageArray ) # dimensions of the full image array in memory 
head( KeysOfImages ) # image keys associated with the images in FullImageArray
head( KeysOfObservations ) # image keys of observations to be associated to images via KeysOfImages
```
We can also analyze the images that we'll use in this analysis. 
```
# plot the second band of the third image
causalimages::image2(FullImageArray[3,,,2])

# plot the first band of the first image
causalimages::image2(FullImageArray[1,,,1])
```
We're using rather small image bricks around each long/lat coordinate so that this tutorial code is memory efficient. In practice, your images will be larger and you'll usually have to read them in from desk (with those instructions outlined in the `acquireImageRepFxn` function that you'll specify). We have an example of that approach later in the tutorial. 

## Writing the `acquireImageRepFxn`
One important part of the image analysis pipeline is writing a function that acquires the appropriate image data for each observation. This function will be fed into the `acquireImageRepFxn` argument of the package functions. There are two ways that you can approach this: (1) you may store all images in `R`'s memory, or you may (2) save images on your hard drive and read them in when needed. The second option will be more common for large images. 

You will write your `acquireImageRepFxn` to take in two arguments: `keys` and `training` 
- `keys` is a character or numeric vector. Each value of `keys` refers to a unique image object that will be read in. If each observation has a unique image associated with it, perhaps `keys = 1:nObs`. In the example we'll use, multiple observations map to the same image. 
- `training` specifies whether to treat the images as in training mode or inference mode. This would be relevant if you wanted to randomly flip images around their left-right axis during training mode to prevent overfitting.

### When Loading All Images in Memory 
In this tutorial, we have all the images in memory in the `FullImageArray` array. We can write an `acquireImageRepFxn` function like so: 
```
acquireImageRepFromMemory <- function(keys, training = F){
  # here, the function input keys 
  # refers to the unit-associated keys 
  return( FullImageArray[match(keys, KeysOfImages),,,] )  
}

ImageSample <- acquireImageRepFromMemory(sample(KeysOfObservations,10))
dim( ImageSample )

# plot image: it's always a good idea 
# to check the images through extensive sanity checks
# such as your comparing satellite image representation
# against those from OpenStreetMaps or Google Earth. 
image2( ImageSample[3,,,1] )
```

### When Reading in Images from Disk 
For most applications of large-scale causal image analysis, we won't be able to read whole set of images into `R`'s memory. Instead, we will specify a function that will read images from somewhere on your harddrive. You can also experiment with other methods---as long as you can specify a function that returns an image when given the appropriate `imageKeysOfUnits` value, you should be fine. Here's an example of an `acquireImageRepFxn` that reads images from disk: 
```
acquireImageRepFromDisk <- function(keys,training = F){
  # IMPORTANT! This is illustration code only; it is not designed to run on your local computer 
  # initialize an array shell to hold image slices
  array_shell <- array(NA,dim = c(1L,imageHeight,imageWidth,NBANDS))

  # iterate over keys:
  # -- images are referenced to keys
  # -- keys are referenced to units (to allow for duplicate images uses)
  array_ <- sapply(keys,function(key_){
    # iterate over all image bands (NBANDS = 3 for RBG images)
    for(band_ in 1:NBANDS){
      # place the image in the correct place in the array
      array_shell[,,,band_] <-
        (as.matrix(data.table::fread( # note the use of data.table::fread to speed up reading in image to memory
          input = sprintf("./Data/Uganda2000_processed/GeoKey%s_BAND%s.csv",
                          key_,
                          band_),header = F)[-1,] ))
    }
    return( array_shell )
  },
  simplify="array")  #using simplify = "array" combines images slices together

  # convert images to tensorflow array for further processing
  # note: your acquireImageRepFxn need not return tensorflow arrays. 
  # R arrays are fine (with dimensions c(nBatch, imageWidth, imageHeight,nChannels)
  # (R arrays will be detected converted and converted internally)
  array_ <- tf$squeeze(tf$constant(array_,dtype=tf$float32),0L)
  array_ <- tf$transpose(array_,c(3L,0L,1L,2L))
  return( array_ )
}
```
## Analyzing the Sample Data 
Now that we've established some understanding of the data and written the `acquireImageRepFxn`, we are ready to proceed with the initial use of the causal image decomposition. 

*Note: The images used here are heavily clipped to keep this tutorial fast; the model parameters chosen here are selected to make training rapid too. The function output here should therefore not be interpreted too seriously.* 

```
ImageHeterogeneityResults <- AnalyzeImageHeterogeneity(
          # data inputs
          obsW =  obsW,
          X = X, # used only if orthogonalize=T
          obsY = obsY,
          imageKeysOfUnits =  KeysOfObservations,
          acquireImageRepFxn = acquireImageRepFromMemory,
          conda_env = "tensorflow_m1", # change "tensorflow_m1" to the location of your conda environment containing tensorflow v2 and tensorflow_probability, 
          
          # inputs to control where visual results are saved as PDF or PNGs 
          # (these image grids are large and difficult to display in RStudio's interactive mode)
          plotResults = T,
          figuresPath = "~/Downloads",
          printDiagnostics = T,
          figuresKey = "CausalImagesTutorial",
          
          # optional arguments for generating transportability maps 
          # here, we leave those NULL 
          transportabilityMat = NULL, # 
          lat =  NULL, # required only if transportabilityMat specified 
          long =  NULL, # # required only if transportabilityMat specified 

          # other modeling options
          orthogonalize = F,
          modelType = "variational_minimal",
          kClust_est = 2,
          nMonte_variational = 10L,
          nSGD = 400L, # make this larger for real applications
          batchSize = 22L,
          channelNormalize = T,
          compile = T,
          yDensity = "normal",
          kernelSize = 3L, maxPoolSize = 2L, strides = 1L,
          nDepthHidden_conv = 2L, # in practice, nDepthHidden_conv would be more like 4L 
          nFilters = 64L,
          nDepthHidden_dense = 0L, nDenseWidth = 32L,
          nDimLowerDimConv = 3L,
          reparameterizationType = "Flipout")
```
## Visual Results 
Upon completion, `AnalyzeImageHeterogeneity` will save several images from the analysis to the location `figuresPath`. The `figuresKey` will be appended to these images to keep track of results from different analyses. Currently, these images include the following: 
-- The image results with .pdf name starting, `VisualizeHeteroReal_variational_minimal_uncertainty`, which plots the images having great uncertainty in the cluster probabilities. -- The image results with .pdf name starting, `VisualizeHeteroReal_variational_minimal_mean_upperConf`: these plots display the images having the highest and lowest lower confidence bound for the different cluster probabilities. Some images may be present multiple times if many observations map to the same image (the computation of the confidence bounds is itself stochastic, so things may not be ordered precisely from run to run). 
-- The image results with .pdf name starting, `VisualizeHeteroReal_variational_minimal_mean`; these plots display the images having the highest probabilities for each associated cluster. 
-- Finally, one output `.pdf` has name starting  `HeteroSimTauDensityRealDataFig`, and plots the estimated distributions over image-level treatment effects for the various clusters. Overlap of these distributions is to be expected, since the quantity is computed at the image [not some aggregate] level.

*Note that, for satellite data, images that show up as pure dark blue are centered around a body of water.*

## Numerical Results
We can also examine some of the numerical results contained in the `ImageHeterogeneityResults` output. 
```
# image type treatment effect cluster means 
ImageHeterogeneityResults$clusterTaus_mean

# image type treatment effect cluster standard deviations
ImageHeterogeneityResults$clusterTaus_sd

# per image treatment effect cluster probability means 
ImageHeterogeneityResults$clusterProbs_mean

# per image treatment effect cluster probability standard deviations
ImageHeterogeneityResults$clusterProbs_sd
```

# Future Development Plan
We now have in beta release code for interpretably decomposing treatment effect heterogeneity by image. In the next stage, we will implement two more functionalities: (1) confounder adjustment via image and (2) causal image system simulation. Core machine learning modules are written in `tensorflow+tensorflow_probability`; subsequent versions may be transfered over to `equinox+oryx+jax`. 

We are committed to the long-term development of this repository and welcome community involvement. 

# Acknowledgements
We thank James Bailie, Cindy Conlin, Devdatt Dubhashi, Felipe Jordan, Mohammad Kakooei, Eagon Meng, Xiao-Li Meng, and Markus Pettersson for valuable feedback on this project and software. We also thank Xiaolong Yang for excellent research assistance.

# References
Connor T. Jerzak, Fredrik Johansson, Adel Daoud. Image-based Treatment Effect Heterogeneity. *Forthcoming in Proceedings of the Second Conference on Causal Learning and Reasoning (CLeaR), Proceedings of Machine Learning Research (PMLR)*, 2023. [`arxiv.org/pdf/2206.06417.pdf`](https://arxiv.org/pdf/2206.06417.pdf)

Connor T. Jerzak, Fredrik Johansson, Adel Daoud. Integrating Earth Observation Data into Causal Inference: Challenges and Opportunities. *ArXiv Preprint*, 2023. [`arxiv.org/pdf/2301.12985.pdf`](https://arxiv.org/pdf/2301.12985.pdf)
