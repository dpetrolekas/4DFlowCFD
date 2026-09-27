# 4DFlowCFD-Toolkit
Open-source framework for MRI-informed cardiovascular CFD, providing patient-specific boundary condition implementation, CFD-to-MRI reconstruction and validation of hemodynamic simulations against in vivo 4D Flow MRI.

## Part 1: 4D Flow MRI (DICOM) to HDF5 format
Converts 4D Flow MRI DICOM magnitude and phase data into 4D velocity arrays and stores the processed dataset in HDF5 format. It also includes utility scripts for inspecting DICOM metadata, VENC information and sequence parameters (Based on the 4D FlowNet framework by Ferdian et al. (2020)).

Example code input showcasing a sagittal slice at representative Magnitude and Phase images in the DICOM format of the example thoracic aorta data:
<p align="center">
  <img width="450" height="229" alt="image" src="https://github.com/user-attachments/assets/1daa793e-0d9e-4809-b2c2-3b06c7faf800" />
  <img width="450" height="229" alt="image" src="https://github.com/user-attachments/assets/6f4b2d56-04cd-42ae-af7a-00e6f9021a5d" />
</p>

Example code output showcasing a sagittal slice at representative Magnitude and Phase images in the 4D array format of the example thoracic aorta data:
<p align="center">
  <img width="450" height="362" alt="image" src="https://github.com/user-attachments/assets/1d946e64-fe87-48dc-9b41-49526fe154df" />
  <img width="450" height="362" alt="image" src="https://github.com/user-attachments/assets/a6abc58c-4b49-4e01-896c-c0bd57e38491" />
</p>

## Part 2: Patient Specific Velocity Profile Extraction and Application
Before any velocity data is extracted, the CFD inlet-cap coordinates are aligned with the MRI coordinate system: the CFD point corresponding to the MRI origin (0,0,0) is identified and subtracted from all cap points to align the origins, a coordinate transformation is applied to express the cap in the MRI reference frame and the coordinates are scaled to match the MRI grid dimensions. This aligned, scaled cap is what later serves as the target for interpolation. 

The 4D Flow MRI velocity data is loaded from the HDF5 file and a Grid of Interest (GOI) is manually defined around the CFD inlet region to limit processing to the relevant MRI volume. For every timestep of the cardiac cycle, the U, V, W velocity components are extracted at each GOI grid point.

These velocities are then spatially interpolated onto the CFD inlet-cap points: the MRI GOI points serve as input points, the CFD cap points as targets, using linear interpolation with nearest-neighbour as a fallback so every inlet point gets a value. This is repeated per timestep.

The coordinates are then transformed back to the original CFD system (velocities unchanged): reverse the scaling, apply the inverse transform (if needed) and restore the original origin offset.

Finally, the boundary-condition (BCT) file is built: inlet points/IDs are taken from the existing bct.dat, matched to the interpolated data by coordinate tolerance and assigned their U, V, W values per timestep . Velocities are converted from m/s to cm/s (if needed), then corrected to the CFD vector convention, the complete patient-specific inlet velocity boundary condition, ready for CFD use or optional temporal refinement (Smoothing).

Example code output: (a) Patient-specific velocity profile in selected reference time points, (b) Inflow waveform, (c) Reference parabolic profile
<p align="center">
  <img width="929" height="581" alt="image" src="https://github.com/user-attachments/assets/696f169a-3a31-4011-87e1-67cf247140c8" />
</p>

## Part 3: CFD Results to Original 4D Flow MRI data Comparison
This step prepares CFD results for direct comparison with 4D Flow MRI data. First, the CFD results are converted into an MRI-compatible format by extracting the X, Y, Z coordinates and U, V, W velocity components from all CFD VTU files and saving them separately for each timestep. A regular 3D grid is then created using the spatial limits and resolution of the 4D Flow MRI data and each CFD point is assigned to its nearest grid point. The velocity fields, maximum velocity values, grid spacing and initial mask are stored in an HDF5 dataset with a structure compatible with the MRI data. The velocity magnitude is calculated for every timestep and a binary CFD mask is created by assigning 0 to zero-velocity points and 1 to non-zero-velocity points. The timestep-specific masks are averaged over the complete cardiac cycle to create a time-averaged CFD mask, which replaces the original mask in the main HDF5 file. The CFD velocity components and their corresponding maximum values are then converted from cm/s to m/s to match the MRI data. The resulting HDF5 dataset contains the CFD velocity fields, maximum velocity values, grid information and final time-averaged CFD mask.

The CFD dataset is then prepared for spatial downsampling. This method is based on the 4D FlowNet code by Ferdian et al. (2020). The complete HDF5 dataset is separated into individual timestep HDF5 files containing the velocity fields, maximum velocity values, mask and grid information. For each timestep, the U, V and W velocity components are processed using FFT-based spatial downsampling with the specified VENC, SNR and downsampling factor. A simulated magnitude image is generated from the CFD mask and complex Gaussian noise is added according to the selected SNR. The CFD mask is also downsampled to the same spatial resolution. The resulting downsampled velocity fields, VENC, SNR and mask from all timesteps are then combined into a single HDF5 dataset. This produces a CFD dataset with a spatial resolution and simulated MRI signal characteristics suitable for comparison with the 4D Flow MRI data.

Example code output: Reconstructed CFD data as downsampled four-dimensional arrays (right) compared to in-vivo 4D Flow MRI (left).
<p align="center">
  <img width="444" height="390" alt="image" src="https://github.com/user-attachments/assets/12ccecca-d82e-4ff1-a7ce-5c2501aad974" />
  <img width="463" height="390" alt="image" src="https://github.com/user-attachments/assets/8b3d2880-ea8f-41e2-8c29-eb912989a690" />
</p>

Finally, representative velocity profiles are extracted from both the original patient 4D Flow MRI data and the downsampled CFD results. For the patient data, the required axial slice is selected and an elliptical ROI is applied to calculate the mean u, v and w velocity components for every timestep. The original velocity components are transformed to the CFD coordinate system and the resulting profiles are saved in a tab-separated text file. The corresponding slice and elliptical ROI are then applied to the CFD data to obtain the mean u, v and w velocity components for every timestep, which are saved using the same data format. When required, the original profiles are linearly interpolated to match the CFD temporal resolution. Separate comparison plots are generated for the u, v and w components, allowing the original and CFD velocity profiles to be compared over the complete cardiac cycle. The maximum and minimum values of each velocity component are determined for both datasets and the corresponding percentage differences are calculated.

Example final comparisons: (a) Predefined ROIs used for velocity comparison. Velocity magnitude distributions in (b) the ascending aorta, (c) proximal to the left common carotid artery (LCCA) branch, (d) proximal to the left subclavian artery (LSA) branch and (e) the upper descending aorta. For each location: (i) in vivo (4D Flow MRI) data, (ii) patient-specific CFD inlet profile and (iii) parabolic CFD inlet profile.

<p align="center"> 
  <img width="940" height="783" alt="image" src="https://github.com/user-attachments/assets/2788905c-93d1-4217-be7c-d8a9daafc202" /> 
</p>

## License
This repository contains both original code and code derived from third-party software, which are subject to same licenses.
