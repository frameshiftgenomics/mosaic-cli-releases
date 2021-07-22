# Mosaic CLI 

## Basic Usage

Typing `mosaic` at the command line will yield a prompt like this:

```
$ mosaic

mosaic-*.*.*

Usage:
  mosaic bam [options] <sample-id> <BAM-or-CRAM>
  mosaic vcf [options] <VCF>
  mosaic gather [options] <dir>...
  mosaic init [options] <project-name>
  mosaic post [options] <project-id> <gathered-mosaic-json>
  mosaic map-paths <gathered-mosaic-json> <map-file></map-file>
  mosaic experiment [options] <project-id> <experiment-name> 

Run each command without any arguments (e.g. "mosaic bam") to see what [options] are available.
```

The philosophy of the CLI is to be easily usable with Unix-style tools and shell scripts, and to not attempt to parallelize or thread anything on its own, so as to easily fit in a variety of compute environments (e.g. batch, slurm, multicore EC2, etc.). 

A simple example with 1 bam and 1 vcf looks like the following:

`$ mosaic bam -d /data/bams NA21129 http://s3.amazonaws.com/1000genomes/phase3/data/NA21129/alignment/NA21129.chrom11.ILLUMINA.bwa.GIH.low_coverage.20121211.bam`

`$ mosaic vcf -d /data/vcfs -s NA21129 http://s3.amazonaws.com/frameshift1000g/all.vcf.gz` (note: this VCF is large, so running will take a while. if you just want to test, I suggest just pulling out a subset first or using a smaller VCF)

`$ mosaic gather /data/bams /data/vcfs`

The above yields a file `gathered.mosaic.json.gz`, which is what we will POST to Mosaic.

Next, we set connection parameters for posting to a running Mosaic instance.
`$ export MOSAIC_URL=https://0.0.0.0:3000/`
`$ export MOSAIC_USERNAME=slichtenberg@frameshift.io`
`$ export MOSAIC_PASSWORD='my$password'`

Alternatively, when using `mosaic post` you can set `MOSAIC_TOKEN` instead of `MOSAIC_USERNAME` and `MOSAIC_PASSWORD`.
The token is an auth token that is provided by Mosaic.
`$ export MOSAIC_URL=https://0.0.0.0:3000/`
`$ export MOSAIC_TOKEN=be0ab7114beb70c14832a0c8de8ed708f2c1a5b8`

Now, we create a new project on Mosaic (this step is optional; the project can already exist). The `init` command returns a project id for us to use if the creation was successful.

```
$ mosaic init my_project
project_id 2
```

Now, we post the data to this project: 
`$ mosaic post 2 gathered.mosaic.json.gz`

And we're done!

## Details / Common issues
+ You'll need to set `LD_LIBRARY_PATH=<your-htslib-1.10-path>`. htslib should be built with `./configure --enable-libcurl` if you need to access AWS files.
+ If using BED files, check that the chromosome names match. For instance, if the BED file has `chr1` but your input files have `1`, the CLI will error out. 
+ The `-d` flag passes an output directory.
+ The use of `http://s3.amazonaws.com/` instead of `s3://` tells the tool to not try to use private credentials, and call the `aws` tool with `--no-sign-request`. This is an unfortunate artifact of the fact that samtools, bcftools, and aws have different conventions for how they handle public S3 files. In the future, this might be accomplished with a CLI flag instead.
+ `-s NA21129` tells `mosaic vcf` to only extract the one sample from the VCF file.
+ Make sure to single-quote your password to prevent the shell from expanding any special characters.


## More advanced usage

A more real-world use case with many samples and files might involve writing fuller bash scripts. The following example was used to produce the BAM output for 1000G files on a 96-core machine, the paths to which were provided in an index file `1000G.index2`.

```
bucket="http://s3.amazonaws.com/1000genomes/phase3/"

# get just the bam column
tail -n +2 1000G.index2 | awk '{ print $1 }' > bam_files.txt

# do it in batches of 96.
ncores=96
while read line; do
      ((i=i%ncores)); ((i++==0)) && wait

      fpath="$bucket$line"
      sample_id=$(echo $line | cut -d '/' -f 2)

      mosaic bam -d /data/bam $sample_id $fpath &
done < bam_files.txt

rm bam_files.txt
```

## S3 Files
In addition to local files, Mosaic CLI works with files stored on S3. If the S3 bucket is public and does not require
credentials to access, you can format the URLs as http://s3.amazonaws.com/\<bucket-name>/\<file-name> and then use the CLI
as if you were working with local files. If the bucket is private, you must set the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
environment variables in order for Mosaic CLI to access the files. In order for Mosaic to be able to access the files, you must
include the -i flag when running `mosaic post`. This will attach the S3 credentials in the request that gets sent to Mosaic.

Example:
```
export MOSAIC_URL=https://0.0.0.0:3000/
export MOSAIC_TOKEN=authtoken123
export AWS_ACCESS_KEY_ID=test123
export AWS_SECRET_ACCESS_KEY=test123
mosaic init test_project
mosaic bam -r 37 sample_id s3://bucket/file.bam
mosaic gather .
mosaic post -i project_id gathered.mosaic.json.gz
```

## S3 compatible object storage

In addition to normal S3 files, Mosaic CLI works with S3 compatible object storage files.
For Mosaic CLI to function correctly with these files, you will need to pass additional options to `mosaic bam`, `mosaic vcf`, and `mosaic post`.
For `mosaic bam` and `mosaic vcf` you will need to pass the -e and -p params which are the S3 endpoint URL and S3 profile respectively
while for `mosaic post` just the -e param needs to be passed.
In order to use these options correctly, the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY environment variables
must be set and a file, ~/.s3cfg, must also exist. The .s3cfg file should be formatted as shown below:
```
[test_profile]
access_key = ****
secret_key = ****
host_base = https://test.com/
```

In this case, the -e option would be `https://test.com/` and the -p option would be `test_profile`

Example:
```
export MOSAIC_URL=https://0.0.0.0:3000/
export MOSAIC_TOKEN=authtoken123
export AWS_ACCESS_KEY_ID=test123
export AWS_SECRET_ACCESS_KEY=test123
mosaic init test_project
mosaic bam -r 37 -e https://test.com/ -p test_profile sample_id https://test.com/file.bam
mosaic gather .
mosaic post -e https://test.com/ -i project_id gathered.mosaic.json.gz
```

## Experiments

The CLI allows users to create experiments for a project by using the `mosiac experiment` command. This will return an
experiment's ID which you can then use in `mosaic bam` and `mosaic vcf` with the -x parameter. An example is below

```
EXP_ID=$(mosaic experiment 1 test-experiment | cut -d ' ' -f 2)
mosaic bam -x $EXP_ID sample_id bam_url
```

## Dependencies

You need at least `samtools` and `bcftools` executables, which the CLI calls out to. You will
also need htslib version 1.10 which is used by mosdepth.

If you're dealing with S3 files, you need the `aws` executable as well.


## Building from source (for dev team)

1. Build our local fork of mosdepth. Move the binary to `bin/mosdepth`.
2. Run `go generate`. This will cause fileb0x to produce a `static` folder.
   If this step does not work, you might need to update Go. Try a version >= go1.13.
3. Run `go build`. This will produce a `mosaic` executable. 


