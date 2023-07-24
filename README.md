# dolphot
This readme will outline the necessary steps to run dolphot for JWST images with any detector. All steps should be run in the terminal.
First, begin by downloading dolphot: http://americano.dolphinsim.com/dolphot/
Download the dolphot2.0 base sources, and then download the MIRI,NIRCAM and NIRISS sources. Additionally download the manual for clarity. Once all the tar files are downloaded and unpacked, place all 3 detector subdirectories into the dolphot2.0 overall directory using, for example:
```
mv ~/dolphot2.0_2/miri ~/dolphot2.0
```
Ensure al subdirectories for the detectors are in the larger dolphot directory.
Uncomment all 3 definitions for MIRI,NIRCAM and NIRISS in the makefile by simply opening it and using a preferred text editor.
Now, run the make file from the dolphot2.0 subdirectory simply by running make. Everything should run without errors. If something fails, check the manual or reach out to Andy Dolphin(he is quite responsive) at adolphin@raytheon.com
Now, that everything is downloaded, you can set the path so that one can properly run dolphot from the terminal. Edit your bashrc file to contain the following:
example:
```
export PATH=~/dolphot2.0/bin:$PATH
```
Now you can begin to work with dolphot. The great advantage of dolphot is that you can run across many images simaultaneously, and dolphot will treat each image separately and give separate results. However, for this example, let us simply assume we are using a single calibrated(level 2) image. Adding images simply requires a tweak to the parameter file. Download images from: https://mast.stsci.edu/portal/Mashup/Clients/Mast/Portal.html
Once you have your images downloaded, you must run the appropriate masking step, which masks pixels or convers them to a usable format for dolphot, depending on the detector. For this step, if one was working with a nircam image which we assume is in some directory on our computer, you would run:
```
ls
jw02767002001_02103_00001_nrcb3_cal.fits
nircammask *.fits >>phot.log
```
Ensure you make a copy of the image, as the masking step alters the image. This would mask the image and then create a log file to keep track of any potential errors and your progress.
Following this, it is best practice to ensure that there is only one science chip you will be running dolphot on(dolphot will not run on multi-chip images-you need to separate and run on each chip) by running splitgroups
```
splitgroups *.fits >>phot.log
```
Now, we must create a sky map for dolphot to use for each image and each chip. To do this, we run calcsky, which uses multiple parameters:

- rin = 10
- rout = 25
- step = 4 
- sigmalow = 2.25
- sigmahigh = 2.00

The manual outlines what exactly each of these parameters does, but essentially dolphot will create a sky map based on an annuus between rin and rout pixels from the pixel whose value is being measured, going in steps based on the step parameter.The algorithm is a mean with iterative rejection. Each iteration, the mean and standard deviation are calculated, and values falling more than σlow below or σhigh above
the mean are rejected. This procedure continues until no pixels are rejected.
```
calcsky
Usage: calcsky <fits base> <inner radius> <outer radius> <step> <lower sigma> <upper sigma>
calcsky jw02767002001_02103_00001_nrcb3_cal 10 25 4 2.25 2.00 >>phot.log
ls
jw02767002001_02103_00001_nrcb3_cal.fits jw02767002001_02103_00001_nrcb3_cal.chip1.fits jw02767002001_02103_00001_nrcb3_cal.chip1.sky.fits
```
Now we can proceed to running the multi-pass photometry stuff. However, we must ensure we have the proper PSFs for dolphot to read to do PSF photometry. We downloaded the psfs again from http://americano.dolphinsim.com/dolphot/ with the appropriate psfs under the detector heading. Once you have unpacked the PSF tar file, move the psfs in a step like this for all revlevant filters you will use:
```
mv ~/F150W.*.psf ~/dolphot2.0/nircam/data
```
Now, we can run dolphot, but we must set key parameters in the dolphot.param file(contained in the param subdirectory, but this should be moved to the directory you run photometry from with all relevant files). The recommended parameters are laid out in the manual, and there are many choices one can make. The first ap-rt fo the parameter file specifices the number of science images one wants to do photometry as well as potentially adding a reference(drizzled) image. We have one image for this example, so this is simple
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
img_apsky = 30 50       #sky annulus for aperture correction
#
```
The comments on the side provide the defintion of each parameter one can choose. Preferred parameters are laid out in each detector's manual. Other parameters in the file are used for finding and measuring stars. DOlphot will run across the entire image, so this is why it is generally used for multi-siource imaging. Of course it can still be used for a single source, but it may take longer than other single-source algorithms.
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
After changing all relevant parameters and setting up the PSF, we can run the true photeomtry dolphot step.
```
dolphot FILE_NAME.phot -pdolphot.param
```
This will create a photometry file with many columns. Luckily, running dolphot creates a variety of files which give the column names and explain the output. In essense, the output file will contain x and y positions of the detections as well as Vega mangitudes. In order to pinpoint what the output for your source is, a bit of additional care is required. One must use the RA/DEC of the source to fin d detections within some small(5-10 pixel) radius around your source. there may be 3-4 detections, but using the following notebook(which has a SN as it's desired source) one can overlay the detections onto the image and tell which magnitude corresponds to the source itself and not some nearby object. Here is an example for SN 2022riv which can be easily duplicated: https://github.com/18rway/dolphot/blob/main/Dolphot_output.ipynb
