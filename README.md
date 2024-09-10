# snakemake-cellprofiler

A thin wrapper around cellprofiler to simplify deploying pipelines on HPC systems.
Cellprofiler solves the problem of reproducible image analysis pipelines, but
when scaling to hundreds of images on slurm systems, additional work is required.
Snakemake is a workflow management system that can dynamically create rules
with specified resources.  Together they can make running pipelines as easy as
changing a few config variables.

The workflow is intended for user with cellprofiler pipelines, lots of data,
and some HPC experience.

## Usage

Install [mamba](https://github.com/conda-forge/miniforge?tab=readme-ov-file#install)
which is recommended for snakemake.

Create a snakemake environment, using version 7
```bash
mamba create -c conda-forge -c bioconda -n snake snakemake=7.32.4
```
Installation should take a few minutes.

Now you are ready to configure your run.  Fill out the following in `config/config.yaml`
```yaml
workdir: "/path/to/output/directory"
# set to a local cp installation if you need extra dependencies
# if unset, will use cellprofiler 4.2.6
cp_env: "cellprofiler"
# set to plugin directory
# if unset, will not include additional plugins
cp_plugin_dir: "/path/to/cellprofiler_plugins/"

# these can be left alone.  If you change, be sure to include the same wildcards
paths:
  config: "logs/config_{date}.yaml"
  slurm_output: "logs/slurm"

  batch_inputs: "{pipeline}/{image_set}/inputs/batch_{batch}.txt"
  batch_outputs: "{pipeline}/{image_set}/temp_outputs/batch_{batch}"
  final_outputs: "{pipeline}/{image_set}/outputs"

# this defines how to run each pipeline
pipelines:
  # these are the default resources for all jobs.  Can override below
  defaults:
    max_batch_size: 0.75  # total image size for batches, in GB
    # resources for cluster integration
    runtime: 63  # in minutes, keep >= 63
    mem_mb: 24000  # in MB
    continue_on_error: False  # if true, will skip images with errors
    recursive_images: False  # if true, include images in all subdirectories

  morphology:  # name replaces {pipeline} above
    pipeline_file: nucleolar_morphology.cppipe  # this is relative to the working directory
    mem_mb: 4000  # this overrides default memory for all image sets in this pipeline
    input_images:
      # name replaces {image_set}.  Location should have a {sample} wildcard
      set1:
        files: /scratch/gpfs/and/remember/{sample}.nd2
      set2:
        mem_mb: 16000  # this overrides the 4000 above
        files: /scratch/gpfs/{sample}.nd2
      set3_or_another_name:
        mem_mb: 24000  # in MB
        runtime: 120  # in minutes, keep >= 63
        files: /scratch/gpfs/another/dir/{sample}.nd2
  another_pipeline:  # can define multiple pipelines
    pipeline_file: another_pipeline.cppipe
    input_images:
      # name replaces {image_set}.  Location should have a {sample} wildcard
      set2:  # can reuse names and paths with different pipelines
        mem_mb: 16000  # in MB
        files: /scratch/gpfs/{sample}.nd2
```

As you can see, resources and pipeline settings can be set as defaults, per
pipeline, or per same set.

To test the resources you estimated, run
```bash
conda activate snake
snakemake --profile cluster test_resources
```
This will run the first 3 batches of each set.  You can examine the resource usage
with `reportseff` in the `logs/slurm` output directory.

**IMPORTANT** If you change the batch_size, you should delete the output files from
your test run!

To run the entire analysis, run `snakemake --profile cluster`.  If some jobs
fail, you can increase the runtime or memory and restart.  As long as the
batch size isn't changed, no files will be recreated.  It's a good idea to
run in a tmux session!

Outputs will be placed into `{pipeline}/{image_set}/outputs` where all batch
outputs are combined into csvs.

## Additional details
The workflow uses fairly basic snakemake features, with the most complex logic
being in parsing the config and updating parameters in a cascading style.
Image batches are dynamically created based on the maximum image size.  Briefly,
images are sorted by size and added to a list of batches.  If adding an image to
the batch would cause it to be larger than the maximum, the image is added to the
next batch.  Snakemake rules are dynamically created with specified resources,
pipelines, and a name composed from the pipeline and image set names.

The workflow has only been tested on linux systems with a slurm resource manager.
Runtimes vary based on pipeline complexity, number and size of images, and the
resources available for execution.

While feature complete, the code is under development and requests for new features
will be considered.  Contributions are also welcome.
