PCB2
=============

Workflow and scripts to support printed circuit board (PCB) milling with the Roland MDX-40A.

##Goal

We are trying to quickly prototype single and double layer PCBs using a CNC mill and software tools.  This repo uses a software tool chain composed of:
 - Eagle (Version 7.0.0 Lite)
 - PC-Board Cam (Version 8/10/2014)
 - Roland VPanel for MDX-40A
 - Custom Python scripts written by DAZL makers.

##Preparation

**MDX-40A**

1. Turn on the MDX-40A and open V-panel.  Some configutration is required before commencing to work through the tool chain.
2. Ensure that coordinate system G54 is selected.
3. Load the 1/64" square end mill and set the zero height using the touch sensor.
4. Click "Setup..." on VPanel.  
 - Ensure that "NC Code" is selected. 
 - Click on "NC Code Setting...".  Here you want to make sure that tab "Tool-diameter Offset" > No 1 is set to 1.57mm.  This represents the radius of the cutter used for boring holes larger than the biggest drill.  If you are not using the 1/8in square end mill for this task then you will need to change the value of "No 1" since it it the parameter added to scripts for circular interpolation tool diameter offset.  See tool diameter offset in the NC Code Reference for more details.

**Workpiece**

1. Clean the copper surfaces of the PCB blank using sandpaper.   You want to remove any oxidation prior to milling.
2. Optionally, you can tin the surface to make soldering easier after milling.
 - On 11/11 I used plumbers tinning flux applied with a cotton swab and then heated with our heat gun to tin the surface.  My initial impression is that this may have caused more bowing of the board than normal, but I will have to see if I like the soldering surface more than the cleaned copper.
3. Secure the workpiece in the jig.

##Workflow for single sided PCB

1. Produce a PCB design in Eagle using the DAZL parts library whenever possible.  Although Eagle make thousands of parts available through its installed libraries, only the DAZL parts have been "package" and "device" parameters that are designed to be cut with the drills and mills we use use at the DAZL.
2. Load the DAZL design rules.
3. Run Eagle's DRC and correct any errors.  This will ensure the signals, pads and vias in the design meet dimensional constraints that can be cut on the mill.
4. Run Eagle 'Cam Processor' and create two sections. The first section should contain your pads, vias, signal and dimension layers, and needs to be in Gerber-RS274X format.  The second section should contain holes and drills and be stored in Excellon format.  For standardization, I name these two files:
  - `<project path>/signal.gtl`
  - `<project path>/drills.txt` 
5. Once you have created these two sections click on File > Save Job... in the menu bar.  For standarization I use the following file name:
  - `<project path>/eagle.cam`
6. Now you want to output the files by clicking on "Process Job" in the cam processor window.
5. Launch PC-Board Cam and click on File > Open > New Circuit.  Load the `signal.gtl` file you just created in the Eagle Cam Processor.  
  - You may get a pop-up window on the Board dimensions.  Ensure that the check-box "Reframe around existing circuitry with a margin of _ _ _ mm" is checked  and contains the value 10mm.  We use this value becasue coordinate frame G54 we use on the MDX-40 has XY origin at (0,0).  We want the lower left corner of the PCB to be at (10,10) to ensure we have safe tool clearance between the hold-downs on the jig and most Southwest element on the PCB.  
  - Finally, the "Z thickness" block should reflect the thickness of your board.  We started with FR4 material that is 1.6mm thick so this will probably be in the default settings.  You may get a pop-up warning that the Z travel of the tools goes to about 1.9mm, and "do you want to change the depth cut of these tools".  Select "No" because we want the tools to cut all the way through into the sacrificial substrate.
6. Ensure there are no "Disabled" tracks that show up as brown signal paths.  
  - To "Re-activate" these tracks right click and select "Re-enable Track net".  
  - Small pads often go unnoticed as disabled at this stage so look closely.
  - Do NOT re-activate tracks around mounting holes and through-holes where heat sinks are mounted (i.e. for TO-220 packages that are screwed down).
6. For single sided boards consider flipping the file in PC-Board Cam on it's X axix.  This will allow you to install components on the substrate side of the board with through-hole components being soldered on the copper side.  Flipping the board makes neater and easier to solder boards.
7. There is a scaling error in the `drills.txt` file coming out of Eagle's cam processor.  Run the drill scaling command line script on the `drills.txt` output from Eagle.  This will rescale the drill and hole locations to match the dimensions of the signal layer.
8. Use the Open > Drill File menu option to load the modified drill file into PC-Board Cam.  This is not trivial.  If you are asked to reduce the size of some drills to match their pads, choose "No".  If you are asked if this is a valid import, choose, "Yes".
9. To orient the drills and holes to the signal layer select Display > Drills in the menu bar.  Then use the rotation commands in PC-Board Cam to get the drill and hole orientation rotated the same as the signal layer.  During this step the visual display of the drills and holes often goes missing.  To get the visual back click on the "Reframe" button on the menu bar.  If this doesn't bring the holes backinto visibility I have had to start over from step 6 on many occassions.  It eventually works! 
10. Next you will notice that the holes do not lay over their corresponding pads in the signal layer.  Use the reference pads tool (bright pink circle targe in the toolbar) to align the two files.
11. Create the milling paths by reselecting the signal layer (layer #1) and clicking on Machine > Calculate Contours.  this step will take a moment to complete.  I find that I need to select Resolution > Medium in the pop-up window in order to cut paths betwen all the small signals.
12. Hit the "Reframe" button to size the PCB based off the elements in your design.
13. Create a zero reference in the bottom left corner of the PCB by hovering over the "Reframe" button and then selecting the "Origin" tool.  Click "Plot Manually" and then click the bottom left corner of the reframed card.
14. In PC-Board Cam select Parameters > Selected Tools.  Ensure that the options match the tools image in this repo.
15. In PC-Board Cam select Parameters > Output data format.  Ensure the configuration option match the output image in this repo.  Then name the location you will store the milling code file at next to the "Create File" radio button.  For standardization I usually use the following filenames
 - Signal layer: `signal.txt`
 - Drills with #65 bit: `65_drill.txt`
 - Drills with #58 bit: `58_drill.txt`
 - Drills with #44 bit: `44_drill.txt`
 - Outline and circular boring on large holes: `cutline.txt`
16. When things look good use the Maching > Mill... option in PC-Board Cam to output a G-Code file for the engraving layer, usually cut with the 1/64" bit.  You should ensure the radio button in the bottom of this pop-up window is selected for "XY-zero point White Cross".  The G-Code file from this step is the instruction set that will run on the Roland MDX-40A (or any other CNC mill). BUT...the output from PC-Board Cam has two problesm that get corrected in post processing.
  - Integer values will cause errors on the MDX-40A.  Feed rates and X, Y and Z axis coordinates cannot be integer values(i.e. F60 X20 will cause and error).  Integer valued quantities must include a trailing zero(i.e. F60.0 X20.0).
  - When the program is using the 1/8 in cutter for holes larger than the #44 drill the mill must take into account the diameter of the tool.  This is accomplished through tool diameter offsite before any circulare interpolation.
17. Run the post-processing script on the output file to to correct all these coordinates.
18. Load the correct cutting tool into the mill, and reset the zero for the following conditions:
 - Any mill.  The mills are depth controlled and of variable length.  The zero must be reset for each mill that is loaded.
 - First drill bit.  Subsequent drill bits do not require the zero to be reset if they are all loaded where the depth ring is snugly against the collet.
19. Load the script and the correct cutter into the mill and output the program through the mill using V-panel.
20. Repeate this process from step 15 for each milling/drilling operation.
