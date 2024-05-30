# Organizing a forecasting challenge

Here are example GitHub repositories that contain the cyberinfrastructure that supports different forecasting challenges:

-   <https://github.com/eco4cast/neon4cast-ci>
-   <https://github.com/eco4cast/usgsrc4cast-ci>
-   <https://github.com/LTREB-reservoirs/vera4cast>

## Requirements

-   S3 storage bucket with credentialed upload and anonymous download

    -   We use Open Storage Network that is available for U.S. National Science Foundation funded grants. You can use Amazon Web Service or other commercial cloud providers. If you have a virtual machine with a public IP address, you can also host your own S3 storage system using [MinIO](https://min.io)

-   S3 storage bucket with anonymous upload and download (this is the submission bucket)

    -   We use virtual machine with a public IP address that runs MinIO.

    -   If you want to hand out credentials to submitting teams then you can use credentialed upload.

-   GitHub Account

## Tasks

The following tasks that support the Challenge are run in the GitHub repository. Most of the task work using settings in the configuration file and do not need to be modified for a new challenge (except for the target generation, dashboard generation, and baseline generation tasks)

<https://github.com/eco4cast/neon4cast-ci/blob/main/challenge_configuration.yaml>

### Target generation

<https://github.com/eco4cast/neon4cast-ci/tree/main/targets>

The organizers need to write R scripts that accesses the "raw" data (in our case NEON data on NEON's cloud storage) and converts the data to the standardized long-format data table. The conversion process may involve QAQC, aggregation, or subsetting (i.e., 10 minute water temperature at multiple depths to daily mean temperature in the top 1 m). The data table is saved as a csv with a stable name (i.e., phenology-targets.csv) and uploaded to S3 storage.

### Submission Processing

<https://github.com/eco4cast/neon4cast-ci/tree/main/submission_processing>

An R script is provided to download submitted forecast files and move the data into the parquet database on S3 storage. The summaries of the forecasts are generated and an inventory is created during the submission processing step.

### Scoring

<https://github.com/eco4cast/neon4cast-ci/blob/main/scoring>

An R script is provided that calculates the CRPS and log score for forecast with corresponding observations. The script is design to only score forecasts when new data is in the targets file from that `datetime`. This prevent repeatably scoring forecasts when neither the forecast nor the data have changed. Another R script generates inventory of the scores.

### Cataloging

<https://github.com/eco4cast/neon4cast-ci/tree/main/catalog>

An R script generates a catalog of the forecast, summaries, targets, scores, and inventories that follows the [STAC standard](https://stacspec.org/en). The set of JSON files that are generated can be viewed in the STAC Browser

### Communicating (dashboard)

<https://github.com/eco4cast/neon4cast-ci/tree/main/dashboard>

A Quarto dashboard is built each day that contains information on the targets, how to submit forecasts, forecast performance, and catalog access. The dashboard code will need to be updated when applied to new challenges. The neon4cast dashboard is here: \<www.neon4cast.org\>

### Forecast Drivers

<https://github.com/eco4cast/neon4cast-ci/tree/main/drivers>

An R script uses the [gefs4cast package](https://github.com/eco4cast/gefs4cast) to download NOAA weather forecasts for each site in the forecasting challenge. The NOAA weather forecasts to converted into a standardized format and upload to S3 storage. The geographic information about the sites is in the [site list](https://github.com/eco4cast/neon4cast-ci/blob/main/neon4cast_field_site_metadata.csv).

### Baseline forecasts

<https://github.com/eco4cast/neon4cast-ci/tree/main/baseline_models>

R scripts are used to generate climatology and persistence null models that are submitted to the challenge.

### Computational environment

<https://github.com/eco4cast/neon4cast-ci/blob/main/Dockerfile>

A Docker Container is built daily that contains all the packages/software required for the task and additional packages for forecast submitters. It is available through [Docker Hub](https://hub.docker.com/repository/docker/eco4cast/rocker-neon4cast/general)

### Automation

Since the GitHub repo is public, GitHub Actions provides free computational resources using a modestly size machine (currently: 4-cores, 16 GB of RAM, 14 GB disk). The Docker container can be used to execute the tasks on a scheduled timing (called a cron schedule). Each task above is associated a yaml file that defines a workflow in <https://github.com/eco4cast/neon4cast-ci/tree/main/.github/workflows>.

For example the following yaml will run a R script in the targets directory at 20:00 UTC each day

```         
on:
  schedule:
    - cron: '0 20 * * *'
  workflow_dispatch:


name: baseline-daily-forecasts

jobs:
  terrestrial:
    runs-on: ubuntu-latest
    container: eco4cast/rocker-neon4cast:latest
    env:
      GITHUB_PAT: ${{ secrets.GITHUB_TOKEN }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    steps:
      - uses: actions/checkout@v4

      - name: Generate forecasts
        shell: Rscript {0}
        run: |
          source("baseline_models/run_terrestrial_baselines.R")
```

You will need to add your S3 creditials to the [GitHub Secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions) in the repo.

## Supporting functions

The neon4cast package in a separate GitHub repo contains functions that are needed by forecast submitters. It is designed to be installed as an R package (unlike the challenge-ci repo).

<https://github.com/eco4cast/neon4cast>

```         
remotes::install_github("eco4cast/neon4cast")
```

The package contains the following functions:

-   `submit()`: A function to upload a forecast to the submission S3 bucket. It has the S3 bucket information within the function and uses the `forecast_output_validator` function in the `neon4cast` package to valid that the forecast follows the require standards.

-   `noaa_stage2()` and `noaa_stage3()`: functions for access NOAA forecasts.

## References

Thomas, R.Q., C. Boettiger, C.C. Carey, M.C. Dietze, L.R. Johnson, M.A. Kenney, J.S. Mclachlan, J.A. Peters, E.R. Sokol, J.F. Weltzin, A. Willson, W.M. Woelmer, and Challenge Contributors. 2023. The NEON Ecological Forecasting Challenge. Frontiers in Ecology and Environment 21: 112-113 <https://doi.org/10.1002/fee.2616>
