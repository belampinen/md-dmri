# Multidimensional data management (MDM). 

This package contains functions for building the local data structure.
 
We work with a structure often referred to as 's'. This structure has the 
following fields

* s.nii_fn  - full path to a nifti file
* s.xps     - the eXperimental Parameter Structure
* s.mask_fn - (optional) path to a nifti file with a 3D mask
               that determines which parts of the data that contains
               actual data and not just background 

The methods are described briefly [here](methods/README.md), in some more detail in two review articles,<sup>1,2</sup> and more exhaustively in the original publications cited for each method.

## EXPERIMENTAL PARAMETER STRUCTURE (xps)

The xps has the following fields. All are in SI units. Not all fields must
be present. The model code checks that the necessary fields are present.

All parameters in the xps should be are stored as variables of size n x m, 
where n is the number of image volumes and m is the number of parameters. 
For example, a 30 direction DTI set with 6 b0 images would render an xps
with many fields, for example, a 'bt' field of size 36 x 6, where 36 
is the total number of images and 6 are the number of parameters required 
to describe the b-tensor.


## A) General parameters

- n:          number of images/signal values (fourth dimension). All 
              parameters need to have n entries.

- t_ex:       cumulative time from start of the experiment to the time of 
              the excitation pulse

- t_acq:      time from excitation to readout of echo, i.e., k = 0 
              (typically equals te in a SE sequence)

- intention:  a string that describes the intention of the experiment, 
              for example
                - PGSE/DTI
                - DDE/uFA
                - Angular-DDE
                - FEXI

- slice_order: not yet defined 

- a_ind:      Averaging index. After averaging (arithmetic or geometric),
              there will be max(a_ind) number of images left.

- s_ind:      Series index. Refers to data acquired in different series, 
              for example, with different prescans (e.g. gain adjustment).
              Can also index acquisitions with different echo times et c.
              

Potentially but not necessarily automatically calculated:

- b_ind:      Indexes measurements according to total b-values

- bd_ind:     Indexes measurements according to b-anisotropy (b_delta)

- be_ind:     Indexes measurements according to b-asymmetry (b_eta)

- br_ind:     Indexes measurements according to b-tensor rotations











## GUI
The function `mgui` starts a graphical user interface that help reviewing the data quality. The left panel shows a folder structure. When opening a data file named e.g. data.nii.gz and the GUI finds an .mat file named as data_xps.mat, it will automatically load it. Then you can draw an ROI an select different methods for fitting the signal data from the dropdown menu in the 'analysis' panel to the right. 

![Image](http://markus-nilsson.github.io/md-dmri/mgui.png)

## Motion and eddy current correction

Conventional motion correction where all data is registered to a volume acquired with b = 0 s/mm<sup>2</sup> is implemented in `mdm_mec_b0`. This type of correction is not appropriate for high b-value data, where other approached such as extrapolation-based registration should be applied (see [this paper](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0141825)). This is supported in the `mdm_mec_eb` method. For an example, see below

```matlab
% Options
opt = mdm_opt;
 
% Connect to the data
s.nii_fn = fullfile(pwd, 'data.nii');
s.xps = mdm_xps_from_gdir('data_gdir.txt');

% Determine output path
o_path = pwd;
 
% Define which data to include in the reference 
% (only b-values below 1000 s/mm2)
s_ref = mdm_s_subsample(s, s.xps.b <= 1e9, o_path, opt); 
 
% Write the elastix parameter file
p = elastix_p_affine;
p_fn = elastix_p_write(p, 'p.txt');
 
% First run a conventional coregistratin of the reference
s_ref = mdm_mec_b0(s_ref, p_fn, o_path, opt);
 
% Run an extrapolation-based registration
s_registered = mdm_mec_eb(s, s_ref, p_fn, o_path, opt);
```

Now `s_registered` can be used in the subsequent analysis.

## Construction of the xps
The xps holds all relevant experimental information, i.e., how the experiment was performed. All variables are given in SI-units, and have predetermined names according to [this specification](mdm/readme.txt). Please see functions `mdm_xps_*` for help on how to construct an xps. Here we give two examples. First, in the case where a single nii file is analyzed. Secondly, we show an example of how multiple nii files are merged and analyzed.

**Creating an xps based on a single nii file**

If a single nii file is the basis for the analysis, a single path is used to define its location `s.nii_fn = 'nii_filename.nii';`. Each nii file is usually associated with files that contain information about the diffusion encoding directions and b-values that were used. These can be used to automatically generate appropriate xps structures. To do this, call

```matlab 
s.xps = mdm_xps_from_gdir('gdir_filename.txt', b_delta);
``` 
or 

```matlab
s.xps = mdm_xps_from_bval_bvec('bval_filename.bval', 'bvec_filename.bvec', b_delta);
```

depending on what format is used. Note that the files currently do not contain information about the shape of the b-tensor, thus `b_delta` must be specified by the user. Furthermore, the path to gdir.txt and .bval/.bvec files can be generated based on `nii_fn` by calling either `mdm_fn_nii2gdir` or `mdm_fn_nii2bvalbvec`. The resulting xps can be saved by using `mdm_xps_save`. If the name of the .mat-file holding the xps is related to the nii filename as specified in `mdm_xps_fn_from_nii_fn`, the framework will automatically find the xps. 

**Example: xps from single nii file**

This example code requires only that the user defines the filename of the nii to create and save the corresponding xps. Note that this assumes that the gdir.txt or .bval/.bvec files are in the same folder an have standardized names.

```matlab
% Define path to nii file 
s.nii_ fn = 'path_to_nii.nii';

% We assume that the encoding is linear
b_delta   = 1;

% Get path to corresponding gdir, and generate the xps
gdir_fn   = mdm_fn_nii2gdir(s.nii_fn, b_delta);
s.xps     = mdm_xps_from_gdir(gdir_fn, b_delta);

% Or, get path to corresponding bval/bvec, and generate the xps
[bval_fn, bvec_fn] = mdm_fn_nii2bvalbvec(s.nii_fn);
s.xps     = mdm_xps_from_bval_bvec(bval_fn, bvec_fn, b_delta);

% Finally, save the xps.
xps_fn    = mdm_xps_fn_from_nii_fn(s.nii_fn);
mdm_xps_save(s.xps, xps_fn)
```


**Creating an xps based on multiple nii files**

In some cases all the necessary data cannot be acquired in a single series, which results in multiple nii files that need to be combined during the analysis. For example, encoding with linear and spherical b-tensors is currently performed in two separate series, but we wish to analyze them simultaneously. In the case when the analysis relies on multiple nii files we first create partial a cell array of partial s structures, and then call `mdm_s_merge` to merge them all into one. There are also separate functions to merge nii files and xps structures (`mdm_nii_merge` and `mdm_xps_merge`).

**Example: xps from multiple nii files**

This example uses alla the tools form the previous example. It assumes that full filenames are specified to all necessary nii files, and that corersponding gdir.txt or .bval/.bvec files with standardized names are in the same folder (see `mdm_fn_nii2gdir` and `mdm_fn_nii2bvalbvec` for standard names). All necessary nii files and corresponding xps structures are first stored in a cell array of structures. Then the cell array of partial s structures is merged. Note that this example includes only the case where gdir.txt files are used to create the xps, but can be modified to use .bval/.bvec according to the example above.

```matlab
% Create a cell array of s structures.
s{1}.nii_fn = 'nii_filename_1.nii';
s{2}.nii_fn = 'nii_filename_2.nii';
s{3}.nii_fn = 'nii_filename_3.nii';

% Corresponding b-tensor shapes. In this case: spherical, linear, and planar.
b_deltas  = [0 1 -.5];

% Define a name for the merged nii (output)
merged_nii_path = '...\output_directory\';
merged_nii_name = 'merged_nii_name.nii';

% Loop over nii files to create partial xps structures, and store them in the cell array.
for i = 1:numel(file_list)
	gdir_fn  = mdm_fn_nii2gdir(s{i}.nii_fn);
    s{i}.xps = mdm_xps_from_gdir(gdir_fn, [], b_deltas(i));
end

% Merge the s structure, and save the merged nii along with its corresponding xps.mat file.
s_merged = mdm_s_merge(s, merged_nii_path, merged_nii_name);

```

Get acquainted with by xps structure by reading mdm/readme.txt.An extensive description of the code structure is found at http://markus-nilsson.github.io/md-dmri/.  


#References
1. D. Topgaard. Multidimensional diffusion MRI. J. Magn. Reson. 275, 98-113 (2017). [link](http://dx.doi.org/10.1016/j.jmr.2016.12.007)
2. D. Topgaard. NMR methods for studying microscopic diffusion anisotropy. In: R. Valiullin (Ed.) Diffusion NMR in confined systems: Fluid transport in porous solids and heterogeneous materials, New Developments in NMR 9, Royal Society of Chemistry, Cambridge, UK (2017). [link](http://dx.doi.org/10.1039/9781782623779-00226)

