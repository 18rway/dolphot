# DOLPHOT TUTORIAL 
This readme will outline the necessary steps to run dolphot for JWST images with any detector. All steps should be run in the terminal.

First, begin by downloading dolphot: http://americano.dolphinsim.com/dolphot/

Download the dolphot2.0 base sources, and then download the MIRI,NIRCAM and NIRISS sources(they are further down the webpage). The downloads will contain manuals which outline many of the steps I will go through here. Once all the tar files are downloaded and unpacked(using tar -xzvf or equivalent), place all 3 detector subdirectories into the dolphot2.0 parent directory using, for example:

```
mv ~/dolphot2.0_2/miri ~/dolphot2.0
```
Ensure all subdirectories for the detectors are in the larger dolphot directory.
Uncomment all 3 definitions for MIRI,NIRCAM and NIRISS in the makefile by simply opening it and using a preferred text editor and removing the #.
Now, run the makefile from the `dolphot2.0` subdirectory simply by running make. Everything should run without errors. If something fails, check the manual or reach out to Andy Dolphin (he is usually quite responsive) at adolphin@raytheon.com

Now, that everything is downloaded, you can set the path so that one can properly run dolphot from the terminal. If you are using a bourne shell(sh,ash,bash,zsh), edit your bashrc file to contain the following:
```
export PATH=~/dolphot2.0/bin:$PATH
```
If you are using a C shell variant instead, run the equivalent setenv command. Now you can begin to work with dolphot. The great advantage of dolphot is that you can run across many images simultaneously, and dolphot will treat each image separately and give separate results. However, for this example, let us simply assume we are using a single calibrated(level 2 _cal) image. Adding images simply requires a tweak to the parameter file. Download images from: https://mast.stsci.edu/portal/Mashup/Clients/Mast/Portal.html
Once you have your images downloaded, you must run the appropriate masking step, which masks pixels or converts them to a usable format for dolphot, depending on the detector. Ensure you make a copy of the image, as the masking step alters the image. This step will mask the image and then create a log file to keep track of any potential errors in the ouput and your progress. For this step, if one was working with a NIRCAM image which we assume is in some directory on our computer, you would run:
```
ls
jw02767002001_02103_00001_nrcb3_cal.fits
nircammask *.fits >>phot.log
```
Following this, it is best practice to ensure that there is only one science extension(or image chip) you will be running dolphot on(dolphot will not run on multi-chip images-you need to separate and run on each chip) by running splitgroups
```
splitgroups *.fits >>phot.log
ls
jw02767002001_02103_00001_nrcb3_cal.chip1.fits
jw02767002001_02103_00001_nrcb3_cal.fits
```
Now, we must create a sky map for dolphot to use for each image and each chip. To do this, we run calcsky, which uses multiple parameters:

- rin = 10
- rout = 25
- step = 4 
- sigmalow = 2.25
- sigmahigh = 2.00

The manual outlines what exactly each of these parameters does, but essentially dolphot will create a sky map based on an annulus between rin and rout pixels from the pixel whose value is being measured, going in steps based on the step parameter. Each iteration, the mean and standard deviation are calculated, and values falling more than σlow below or σhigh above
the mean are rejected. This procedure continues until no pixels are rejected.
```
calcsky
Usage: calcsky <fits base> <inner radius> <outer radius> <step> <lower sigma> <upper sigma>
calcsky jw02767002001_02103_00001_nrcb3_cal 10 25 4 2.25 2.00 >>phot.log
ls
jw02767002001_02103_00001_nrcb3_cal.fits jw02767002001_02103_00001_nrcb3_cal.chip1.fits jw02767002001_02103_00001_nrcb3_cal.chip1.sky.fits
```
Now we can proceed to running the multi-pass photometry step. However, we must ensure we have the proper PSFs for dolphot to read to do PSF photometry. We downloaded the psfs again from http://americano.dolphinsim.com/dolphot/ with the appropriate psfs under the detector heading. Once you have unpacked the PSF tar file, move the psfs to the parent directory for all relevant filters you will use:
```
mv ~/F150W.*.psf ~/dolphot2.0/nircam/data
```
Now, we can run dolphot, but we must set key parameters in the dolphot.param file(a copy of this file which you can edit is in this repository-place this in the folder you run dolphot from). The recommended parameters are laid out in the manual, and there are many choices one can make. The first part of the parameter file specifies the number of science images one wants to do photometry on as well as potentially adding a reference(drizzled) image. We have one image for this example, so this is simple
```
more dolphot.param
Nimg = 1                #number of images (int)
#
# The following parameters can be specified for individual images (img1_...)
#  or applied to all images (img_...)
img1_file = jw02767002001_02103_00001_nrcb3_cal.chip1            #image 1
img_shift = 0 0         #shift relative to reference
img_xform = 1 0 0       #scale, distortion, and rotation
img_PSFa = 3 0 0 0 0 0  #PSF XX term (flt)
img_PSFb = 3 0 0 0 0 0  #PSF YY term (flt)
img_PSFc = 0 0 0 0 0 0  #PSF XY term (flt)
img_RAper = 2.5         #photometry apeture size (flt)
img_RChi = -1           #Aperture for determining centroiding (flt); if <=0 use RAper
img_RSky = 4 10         #radii defining sky annulus (flt>=RAper+0.5)
img_RPSF = 13           #PSF size (int>0)
img_aprad = 20          #radius for aperture correction
img_apsky = 30 50#sky annulus for aperture correction
photsec= 0 1 490 490 510 510      #region to do photometry on
#
```
The most important parameter to set here(for individual objects) is photsec, which can determine how much of an image you photometer. To determine how much of the image to photometer, simply find the ra/dec of the object and convert it to pixrel values using astropy( first step here https://github.com/18rway/dolphot/blob/main/Reading_dolphot.ipynb) or use ds9 for a rough estimate of the pixel value. Photometer a region that is +/- 20 of the given x and y you find to be safe. So, if the location was (500,500), you would enter photsec= 0 1 490 490 510 510 and dolphot would only do photometry on detections in this region. For an image where you are interested in a variety of objects, this step is not necesarry. The comments on the side provide the defintion of each parameter one can choose. Preferred parameters are laid out in each detector's manual. Other parameters in the file are used for finding and measuring stars.  Of course it can still be used for a single source, but it may take longer than other single-source algorithms.
```
photsec =               #section: group, chip, (X,Y)0, (X,Y)1
RCentroid = 2           #centroid box size (int>0)
SigFind = 2.5           #sigma detection threshold (flt)
SigFindMult = 0.85      #Multiple for quick-and-dirty photometry (flt>0)
SigFinal = 3.0          #sigma output threshold (flt)
MaxIT = 25              #maximum iterations (int>0)
#FPSF = G+L              #PSF function (str/Gauss,Lorentz,Lorentz^2,G+L)
PSFPhot = 1             #photometry type (int/0=aper,1=psf,2=wtd-psf)
PSFPhotIt = 2           #number of iterations in PSF-fitting photometry (int>=0)
FitSky = 2              #fit sky? (int/0=no,1=yes,2=small,3=with-phot)
SkipSky = 1             #spacing for sky measurement (int>0)
SkySig = 2.25           #sigma clipping for sky (flt>=1)
NegSky = 1              #allow negative sky values? (0=no,1=yes)
NoiseMult = 0.10        #noise multiple in imgadd (flt)
FSat = 0.999            #fraction of saturate limit (flt)
#Zero = 25.0             #zeropoint for 1 DN/s (flt)
PosStep = 0.25          #search step for position iterations (flt)
dPosMax = 3.0           #maximum single-step in position iterations (flt)
RCombine = 1.5          #minimum separation for two stars for cleaning (flt)
SigPSF = 5.0             #min S/N for psf parameter fits (flt)
#PSFStep = 0.25          #stepsize for PSF
#MinS = 1.0              #minimum FWHM for good star (flt)
#MaxS = 9.0              #maximum FWHM for good star (flt)
#MaxE = 0.5              #maximum ellipticity for good star (flt)
#
```
After changing all relevant parameters and setting up the PSFs, we can run the photometry step using the dolphot command and specifying the relevant parameter file.
```
dolphot 2022riv_f150w.phot -pdolphot.param
```
The filename can bet set by you. This will create a photometry file with many columns. Running dolphot creates a variety of files which explain the column names(2022riv_f150w.phot.columns) and explain the output. The output file will contain x and y positions of the detections as well as Vega magnitudes. In order to pinpoint what the output for your source is, a bit of additional care is required. One must use the RA/DEC of the source to find detections within some small(5-10 pixel) radius around your source. There may be 3-4 detections, but using the following notebook one can overlay the detections onto the image and tell which magnitude corresponds to the source itself and not some nearby object. Here is an example for SN 2022riv which can be easily duplicated: [Parsing dolphot output](https://github.com/18rway/dolphot/blob/main/Reading_dolphot.ipynb)
