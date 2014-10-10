<h1>FijiBento_Tutorial</h1>

Tutorial explaining how to use the FijiBento suite to perform alignment on Serial Section EM images.

<h2>Requirements:</h2>
- Java (ver - TBD)
- Python (ver - TBD)

There are several jar files that are used by FijiBento's Java implemented parts. We use Maven (http://maven.apache.org) to fetch these jars.

<h2>Data:</h2>
FijiBento uses the Tile-Spec json specification (https://github.com/saalfeldlab/alignment-hackathon-2014-04/wiki/TileSpec) as input and output files.
Each json file should describe a single section. No two files should point to the same section.
Example files are provided at the examples directory (TBD).




<h2>2D alignment (montage):</h2>
2D alignment (section montaging) takes a single Tile-Spec json file as input
which describes a single section that is comprised of several tiles that are
roughly aligned. Each tile should have some overlap with at least another tile.
The alignment process performs a (how does it work - TBD).
Its output is a tile-spec file with a finer alignment of all the tiles into a single section.

How does it work:
TBD

Process phases:
<ol>
<li> Computing SIFT features - we start by computing the SIFT features for each tile in the section.
</li>

<li> Matching SIFT features - TBD
</li>

</ol>


<h2>3D alignment:</h2>
3D alignment takes a directory that has multiple per-section Tile-Spec json files as input, and aligns these sections.
Our tool is based on the alignment algorithm that were introduced in TrakEM2 (http://www.ini.uzh.ch/~acardona/trakem2.html).

</h3>How does it work:</h3>
The goal of the 3D alignment process is to take a series of sections (given in a json format), aligning them, and saving the output as json files.
The tool works in 3 phases: features computation, matching adjacent sections, and optimizing the entire stack.
<br>The first phase computes SIFT features individaully for each section. In the second phase every two adjacent sections are matched.
Not only direct neighboring sections can be matched, but also sections that are "far" from each other.
This can be especially helpful in cases where there are a few errors in the scanned images, but we still look for a good alignment.
The distance between the adjacent sections has an impact on the elastic alignment of these sections, therefore the weight of a match decreases as the distance grows.
<br>The second phase executes starts by matching the SIFT features of two sections, and filtering these matches (using RANSAC http://en.wikipedia.org/wiki/RANSAC) to create some initial model that describes the transformation between the sections. It then creates a mesh of the sections, and executes a Block-Matching algorithm between the two sections.
The last phase performs an optimization on the entire stack of images, and produces a json file for each section with the appropriate elastic transformation.

<h3>Process phases:</h3>

1. Computing SIFT features - we start by computing the SIFT features for the entire section (probably after downsampling the section).
Executed using:
create_layer_sift_features.py tilespec_file_name [-o output_sifts_json_file_name] [-j jar_file_name] [-c conf_file_name] [-t threads_num]
Parameters:
* tilespec_file_name - a tile spec json file to compute the SIFT features for.
* -o output_sifts_json_file_name - the file where the SIFT features will be saved.
* -j jar_file_name - the path to the FijiBento jar file name.
* -c conf_file_name - the configuration file that includes the parameters to the java application.
* -t threads_num - the number of threads to use.


<li> For each two sections that we wish to align together:
<ol>
<li> Matching SIFT features - match the SIFT features of the two sections.
Executed using:
match_layers_sift_features.py tilespec_file_name1 sifts_json_file_name1 tilespec_file_name2 sifts_json_file_name2 [-o output_sift_matches_json_file_name] [-j jar_file_name] [-c conf_file_name] [-t threads_num]
Parameters:
** tilespec_file_name1 - a tile spec json file for the first section.
** sifts_json_file_name1 - a sift features json file for the first section.
** tilespec_file_name2 - a tile spec json file for the second section.
** sifts_json_file_name2 - a sift features json file for the second section.
** -o output_sift_matches_json_file_name - the file where the matches of the SIFT features will be saved.
** -j jar_file_name - the path to the FijiBento jar file name.
** -c conf_file_name - the configuration file that includes the parameters to the java application.
** -t threads_num - the number of threads to use.
</li>

<li> Filter the matching features - to get a model that roughly aligns these sections.
Executed using:
filter_ransac.py sift_matches_json_file_name compared_url [-o output_model_json_file_name] [-j jar_file_name] [-c conf_file_name]
Where compared_url is the url of the first json file
Parameters:
** sift_matches_json_file_name - a correspondence json file for the matchings between the first section and the second.
** compared_url - the url of the tile spec to compare against (typically the url of the second tile-spec from the previous step).
** -o output_model_json_file_name - the file where the transformation model between the two sections will be saved.
** -j jar_file_name - the path to the FijiBento jar file name.
** -c conf_file_name - the configuration file that includes the parameters to the java application.
</li>

<li> Match by Max-PMCC - creates a triangular mesh from both sections, and attempts to match and elastically align each corresponding triangles between the two sections.
Executed using:
match_layers_by_max_pmcc.py tilespec_file_name1 tilespec_file_name2 model_1_to_2_json_file_name -W section_width -H section_height [-o output_pmcc_matches_json_file_name] [-j jar_file_name] [-c conf_file_name] [-t threads_num] [-f fixed_layers] [--auto_add_model]
Parameters:
** tilespec_file_name1 - a tile spec json file for the first section.
** tilespec_file_name2 - a tile spec json file for the second section.
** model_1_to_2_json_file_name - a model json file that contains the model between the two sections (in case the model is empty, one can use --auto_add_model to set the default identity model in this case).
** -W section_width - the width (number of pixels) of a section.
** -H section_height - the height (number of pixels) of a section.
** -o output_pmcc_matches_json_file_name - the file where the matches of the block matching algorithm will be saved.
** -j jar_file_name - the path to the FijiBento jar file name.
** -c conf_file_name - the configuration file that includes the parameters to the java application.
** -t threads_num - the number of threads to use.
** -f fixed_layers - a list of section numbers that are considered to be fixed (will stay as they are), e.g., "1 2 5" indicates that sections 1, 2, and 5 are fixed.
** --auto_add_model - allows using the default identity model when no model is found in the previous steps between the two sections.
</li>
</ol>
</li>

<li> Optimization - Optimizes the entire stack of sections into a single aligned 3D image.
Executed using:
optimize_layers_elastic.py (all_tilespec_file_names | tilespec_file_name1 tilespec_file_name2 ...) (all_pmcc_matches_file_name | pmcc_matches_json_file_name1 pmcc_matches_json_file_name2 ...) -W section_width -H section_height [-o output_dir] [-j jar_file_name] [-c conf_file_name] [-t threads_num] [-f fixed_layers]
Where the output is a list of Section_[layer_num].json files.
Parameters:
** (all_tilespec_file_names | tilespec_file_name1 tilespec_file_name2 ...) - receives either a single file that contains a line-delimeted list of tile-spec json files to parse, or an explicit list of tile spec json files.
** (all_pmcc_matches_file_name | pmcc_matches_json_file_name1 pmcc_matches_json_file_name2 ...) - receives either a single file that contains a line-delimeted list of correspondence json files to parse, or an explicit list of correspondence json files.
** -W section_width - the width (number of pixels) of a section.
** -H section_height - the height (number of pixels) of a section.
** -o output_dir - the directory where the output json files (after alignment) will be saved. The files are saved in the format "Section_012.json", where the number represents the section number.
** -j jar_file_name - the path to the FijiBento jar file name.
** -c conf_file_name - the configuration file that includes the parameters to the java application.
** -t threads_num - the number of threads to use.
** -f fixed_layers - a list of section numbers that are considered to be fixed (will stay as they are), e.g., "1 2 5" indicates that sections 1, 2, and 5 are fixed.
</li>
</ol>

* Running all phases on a single machine:
3d_align_driver.py input_json_files_dir [-w work_dir] [-o output_dir] [-j jar_file_name] [-c conf_file_name] [-d max_layer_distance] [--auto_add_model]
Parameters:
** input_json_files_dir - a directory where a tile-spec json file per section are found.
** -w work_dir - the directory where the various phases json files will be saved.
** -o output_dir - the directory where the output json files (after alignment) will be saved. The files are saved in the format "Section_012.json", where the number represents the section number.
** -j jar_file_name - the path to the FijiBento jar file name.
** -c conf_file_name - the configuration file that includes the parameters to the java application.
** -d max_layer_distance - the maximal distance between two neighboring sections that need to be matched.
** --auto_add_model - allows using the default identity model when no model is found between two matched sections.



* Running all phases on the Odyssey cluster:
3d_align_cluster_driver.py input_json_files_dir [-w work_dir] [-o output_dir] [-j jar_file_name] [-c conf_file_name] [-d max_layer_distance] [--auto_add_model]
Parameters:
** input_json_files_dir - a directory where a tile-spec json file per section are found.
** -w work_dir - the directory where the various phases json files will be saved.
** -o output_dir - the directory where the output json files (after alignment) will be saved. The files are saved in the format "Section_012.json", where the number represents the section number.
** -j jar_file_name - the path to the FijiBento jar file name.
** -c conf_file_name - the configuration file that includes the parameters to the java application.
** -d max_layer_distance - the maximal distance between two neighboring sections that need to be matched.
** --auto_add_model - allows using the default identity model when no model is found between two matched sections.
** -k - Keeps running all the jobs on the cluster, and if one fails or crashes, it is re-executed.
** -m - Aggregates multiple small jobs to a single large job, and executes the large jobs on the cluster.
** -mk - Aggregates multiple small jobs to a single large job, and executes the large jobs on the cluster, while keeping sure that all the jobs are re-executed in case a job fails or crashes.


<h2>Parameter conversion from TrakEM2 to this tool</h2>


* Screen1 - Block Matching Parameters

Block Matching:
layer scale (0.10)      => MatchSiftFeatures.layerScale, FilterRansac.layerScale, MatchLayersByMaxPMCC.layerScale, OmptimizeLayersElastic.layerScale
search radius (200 px)  => MatchLayersByMaxPMCC.searchRadius
block radius (-1 px)   => MatchLayersByMaxPMCC.blockRadius
resolution (16)         => MatchLayersByMaxPMCC.resolutionSpringMesh, OptimizeLayersElastic.resolutionSpringMesh (default: 32)

Correlation Filters:
minimal PMCC r (0.60)                   => MatchLayersByMaxPMCC.minR
maximal curvature ratio (10.00)         => MatchLayersByMaxPMCC.maxCurvatureR
maximal second best r/best r (0.90)     => MatchLayersByMaxPMCC.rodR

Local smoothness filter:
use local smoothness filter (true)                  => MatchLayersByMaxPMCC.useLocalSmoothnessFilter (default: false)
approximate local transformation (Rigid / 1)        => MatchLayersByMaxPMCC.localModelIndex
local region sigma (200.00 px)                      => MatchLayersByMaxPMCC.localRegionSigma
maximal local displacement (absolute) (100.00 px)   => MatchLayersByMaxPMCC.maxLocalEpsilon (default: 12)
maximal local displacement (relative) (3.00)        => MatchLayersByMaxPMCC.maxLocalTrust

Miscellaneous:
layers are pre-aligned (false)      => N/A
test maximally (1 layers)           => N/A (in the driver script)


* Screen2 - SIFT Parameters

Scale Invariant Interest Point Detector:
initial gaussian blur (1.60 px)     => ComputeSiftFeatures.initialSigma
steps per scale octave (3)          => ComputeSiftFeatures.steps
minimum image size (64 px)          => ComputeSiftFeatures.minOctaveSize
maximum image size (1024 px)        => ComputeSiftFeatures.maxOctaveSize

Feature Descriptor:
feature descriptor size (8)                 => ComputeSiftFeatures.fdSize
feature descriptor orientation bins (8)     => ComputeSiftFeatures.fdBins

Local Descriptor Matching:
closest/next closest ratio (0.92)   => MatchSiftFeatures.rod

Miscellaneous:
clear cache (true)                  => N/A
feature extraction threads (4)      => (ComputeSiftFeatures threads)


* Screen3 - Geometric Filters

maximal alignment error (200.00 px)         => FilterRansac.maxEpsilon, OptimizeLayersElastic.maxEpsilon
minimal inlier ratio (0.00)                 => FilterRansac.minInlierRatio
minimal number of inliers (12)              => FilterRansac.minNumInliers
approximate transformation (Affine / 3)     => FilterRansac.modelIndex
ignore constant background (false)          => FilterRansac.rejectIdentity
tolerance (5.00 px)                         => FilterRansac.identityTolerance
give up after (3 failures)                  => FilterRansac.maxNumFailures


* Screen4 - Optimization

Approximate Optimizer:
approximate transformation (Rigid / 1)  => OptimizeLayersElastic.modelIndex
maximal iterations (1000)               => (Unused in the original code)
maximal plateauwidth (200)              => (Unused in the original code)

Spring Mesh:
stiffness (0.10)                => MatchLayersByMaxPMCC.stiffnessSpringMesh, OptimizeLayersElastic.stifnessSpringMesh
maximal stretch (2000.00 px)    => MatchLayersByMaxPMCC.maxStretchSpringMesh, OptimizeLayersElastic.maxStretchSpringMesh
maximal iterations (1000)       => OptimizeLayersElastic.maxIterationsSpringMesh
maximal plateauwidth (200)      => OptimizeLayersElastic.maxPlateauwidthSpringMesh
use legacy optimizer (true)     => OptimizeLayersElastic.useLegacyOptimizer (default: false)


Parameters that were added to FijiBento:
- ComputeSiftFeatures.avoidTileScale (false) -
    instead of using the default computed scale (that is set
    by the maxOctaveSize and the image width), use 1.0.
- MatchLayersByMaxPMCC.autoAddModel (false) -
    Automatically adds identity model when filter ransac cannot
    find a model.
- OptimizeLayersElastic.dampSpringMesh, MatchLayersByMaxPMCC.dampSpringMesh (0.9) -
    This is a value that is found in the original TrakEM2 parameters,
    but is not modified as one of the parameters (but used in the program).

