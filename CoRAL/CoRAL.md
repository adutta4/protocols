# Running CoRAL on the SBP Pines Compute Cluster
Assuming that you have created an account, login to pines using `ssh usernames@pines` and input your password. If you need help with this step, refer to [Owen's Tutorial](https://github.com/chavez-lab/protocols/wiki/The-pines-compute-cluster-at-SBP).

The tutorial is based on the Bafna Lab's [CoRAL Github repo](https://github.com/AmpliconSuitE/CoRAL). 

## Setting up the environment for CoRAL
There are a few steps to complete in order to get set up to run CoRAL.

1. Obtain a [Gurobi license](https://www.gurobi.com/academia/academic-program-and-licenses/). Making an account and downloading an academic license is free. Once you have the license, you can move it to your home directory on pines using: 
    ```bash
    scp gurobi.lic username@pines:/home/username
    ```
2. The easiest way to have everything set up correctly for CoRAL to run is to create a new conda environment and install the required packages. If you don't have conda on pines, you can set it up using [this tutorial](https://github.com/chavez-lab/protocols/tree/main/Setting_up_your_workstation). You can create the new environment with the following commands:
    ```bash 
    conda create -n coral
    conda activate coral
    ```

    On pines, CoRAL is downloaded at `/shares/chavez_lab/expanse/bin/CoRAL`. The packages needed for CoRAL are listed at `/shares/chavez_lab/expanse/bin/CoRAL/conda_requirements.txt`. I would recommend installing each of the lines below individually, as I ran into some unknown problems with package versions when trying to installing them together:

    ```
    conda config --add channels https://conda.anaconda.org/gurobi
    conda install gurobi
    conda install bioconda::cnvkit
    conda install conda-forge::cvxopt
    conda install conda-forge::intervaltree
    conda install matplotlib
    conda install numpy
    conda install pandas
    conda install bioconda::pysam
    conda install bioconda::bioconductor-dnacopy
    ```
Everything is now installed and set up to start running the analysis pipeline. 

 ## Analysis Pipeline
 There are several steps to get from the initial pod5 nanopore files to CoRAL output. A general overview of these steps are: 

1. Run dorado for basecalling - conversion to a BAM file
2. Samtools sort and index 
3. Run CNVkit - cncalls.py, provided in the CoRAL download in order to get the copy number calls that are used by CoRAL.
4. CoRAL seed and reconstruct: this will output the cycle and graph files from CoRAL
5. CoRAL plot: generates visual plots from the cycle and graph files.

Optional following steps for analysis:

1. Run AmpliconClassifier - a tool developed by the Bafna Lab to determine whether a given amplicon is ecDNA, linear, BFB, etc.
2. CycleViz - converts the graph from CoRAL into a circularized view. 
 
## Step 1: Run Dorado basecaller
An example script for this step can be found in `shares/chavez_lab/expanse/projects/nanopore/coral/case65_3a/run_dorado.sbatch` (note: will be moved to scripts directory in the future).

```
dorado=/mnt/beegfs/shares/chavez_lab/expanse/projects/nanopore/nanopore/dorado-0.6.1-linux-x64/bin/dorado
pod5_dir={Directory where your pod5 files are located}
hg38=/mnt/beegfs/shares/chavez_lab/expanse/anno/hg38/hg38.fa

$dorado basecaller --recursive --reference $hg38 "hac,5mCG_5hmCG" $pod5_dir > {Your File Name}.bam
```

To run this script, copy it into the directory you wish to run it in. You will need to change the directory where your pod5 files are stored (`$pod5_dir`) and the file name, as shown above. Submit the job with slurm using `sbatch run_dorado.sbatch`. Once this has run, you should obtain a BAM file for your sample. 

## Step 2: Run Samtools sort and index
The BAM file you obtained above has to be sorted and  indexed before cnvkit can run. The script to do this is located at the same directory as above, called `samtools.sbatch`. You will need to change the `source` to your own conda.sh file (this is likely similar to `/home/{your username}/miniconda3/etc/profile.d/conda.sh`). In addition, the file names can be changed accordingly. 

## Step 3: Get copy number calls

From the sorted BAM file obtained in step 2, you will now need copy number calls before running CoRAL. An example script is located in `/shares/chavez_lab/expanse/projects/nanopore/coral/case68/cncalls.sbatch`. Copy this file to your directory, and change the source. Submit the script using `sbatch cncalls.sbatch {sorted_BAM_file}`. This will create a file called [input].cns, which you can feed to CoRAL for it's --cn_segs argument.

## Step 4: Run CoRAL seed and reconstruct 

Finally, you should have all the files required to run CoRAL. An example script is located at `/shares/chavez_lab/expanse/projects/nanopore/coral/case68/coral.sbatch` (note: this will also be moved to scripts). This runs the two main steps of CoRAL, seed and reconstruct. In addition, within the file you should change `case68` to your own file's basename. To run this script, copy it into your directory again, and change the source. To submit the script, use the format:

`sbatch coral.sbatch {[input].cns} {sorted.bam}`

## Step 5: Run CoRAL plot

Once you have the graph and cycle reconstructions from the previous step, CoRAL also provides functionality to create a plot from the output files in order to improve data visualization. This produces output similar to the plots that AmpliconArchitect provides. An example script is provided at `/shares/chavez_lab/expanse/projects/nanopore/coral/case68/coral_plot.sbatch`. as before, this file can be copied into your directory. The AA-formatted `_graph.txt` file and `_cycles.txt` from step 4 will be the required input for this step, along with your bam file and the output prefix you are using.

Once the file paths in the scripts are modified, you can simply run the script with `sbatch coral_plot.sbatch $GRAPH $CYCLES $BAM $OUTPUT_PREFIX`.

Along with the in-built `plot` functionality in CoRAL, the graph and cycle files are formatted such that they can be input into other AmpliconArchitect-adjacent tools, such as [AmpliconClassifier](https://github.com/AmpliconSuite/AmpliconClassifier) and [CycleViz](https://github.com/AmpliconSuite/CycleViz) as well, if desired.

## Run CycleViz

To run CycleViz, you will need to make a new conda environment for CycleViz. You can do this with the following commands: 

``` 
conda create -n cycle_viz
conda install intervaltree 'matplotlib-base>=2.0.0' numpy pyyaml
```

Next, you can copy the script `run_cycleviz.sbatch` from the case68 folder into your directory. Run this with the command `sbatch run_cycleviz.sbatch $GRAPH $CYCLES`, where `$GRAPH` and `$CYCLES` are the files you generated in Step 4. 