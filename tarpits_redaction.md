## Zooniverse Redaction Project Workflow

Version 1.0, last updated: 2/29/2020.
Current project workflow: ID 13300 v1.1

### Workflow Outline

1. Ensure you have the complete data from the Zooniverse project, including the
classification .csv, the subjects .csv, and the workflow .csvs.
2. If you are running Windows, find and install a Bash command line shell (e.g. Git Bash).
This will make it easier to follow along with the instructions here, which use Unix-style commands.
(If you are using Mac or Linux, your terminal/shell should be fine.)
3. Install python and the `panoptes_aggregation` package.
4. Do the question task extraction and reduction.
5. Using the results from step 4, filter classification data to only include rows
 corresponding to subjects that participants determined should be included in the project.
  - `filter_by_yes_include.R`
6. Do the text task extraction and reduction.
7. Use the de-nesting script twice to de-nest the two combo tasks in the classification data.
  - `remove_combo_nesting.py`
8. Do dropdown task extraction and reduction on the de-nested classification data.
9. Do the shape task extraction and reduction.
10. Prepare a special .csv for the python redaction program.
  - `prepare_for_redaction.R`
11. Import the prepared data into and run the python redaction program, which
will scrape, redact, and save the modified images.
  - `redact_v2.py`
12. **\**** 'Widen' dropdown data and join reduced data with subjects data.
  - `combine_reduced_csvs.R`
13. **\**** Use OpenRefine cleaning process to extract data from awkward structures and to
find & replace codes in the data with paleontologist names & month names.
  - `tarpits_codebook.txt`

**\**** = Optional but Recommended

### From the Zooniverse data dump (1-3)

From the Zooniverse project ["Data Exports" page](https://www.zooniverse.org/lab/9524/data-exports),
download the most up-to-date versions of the three following files:
- classification export
- subject export
- workflow export

You will also need to have the `panoptes_aggregation` python package installed,
which offers a command line interface for aggregating the raw classification data.
If you do not have python installed natively on your machine, you will want to do that
before continuing. You should be able to install the `panoptes_aggregation` package
using your preferred package management tool (pip or conda should both work fine).
I recommend consulting the [aggregation package documentation page](https://aggregation-caesar.zooniverse.org/docs),
though we will also go over how to run the package here. For example, with pip,
you can install the package with the following command:

- Mac/Linux: `pip install panoptes_aggregation`
- Windows: `python -m pip install panoptes_aggregation`

In order to run the `panoptes_aggregation` package from the command line in Bash, it has to know where to “find” the package. The package is located in the Scripts folder, wherever you decided to install Python. If you installed to C:\Python, you can tell Bash where to find the scripts by adding it to PATH:

`export PATH=/c/Python:/c/Python/Scripts:$PATH`

With `panoptes_aggregation` installed and the PATH updated, you should be ready
to start the aggregation process.

### Question extraction & reduction (4)

To begin extracting the question data, you need to confirm the workflow ID. At
the top of the csv titled "tar-pits-project-workflows" you will see columns titled
"workflow_id" "version" "minor_version". Knowing the placement of these columns,
scrolling to the bottom of this .csv you'll find the latest workflow version.
An example would be version 13228 V22.15 in which 13228 corresponds to "workflow_id"
column, 22 to "version" and 15 to "minor_version".

The first part of aggregation is generating configuration files. The general command
is as follows:

`panoptes_aggregation config tar-pits-zooniverse-project-workflows.csv
[workflow_id value] -v [version] - m [minor_version]`

From this point, I will assume the current version as of the time of this document,
so the command would be

`panoptes_aggregation config tar-pits-zooniverse-project-workflows.csv
13300 -v 1 - m 1`

From this, you will get one extractor config .yaml file and four reducer config .yaml files.
Of these you need the extractor config and the reducer config that ends in question_extractor.yaml.

From there, you can extract the .csv using the extractor .yaml file using this command:

`panoptes_aggregation extract tar-pits-zooniverse-project-classifications.csv
Extractor_config_workflow_13300_V1.1.yaml`

Now you can reduce the .csv with the respective reducer .yaml file using this command:

`panoptes_aggregation reduce question_extractor_extractions.csv
Reducer_config_workflow_13300_V1.1_question_extractor.yaml`

This will produce a .csv where subjects will have the number of "Yes, this should be included"
selections in a column called "data.yes", and the number of "No, this should not be included"
selections in a column called "data.no". You will need this file for the next step.

### Filtering classifications (5)

Next, before we aggregate the data for the other three task types, we want to filter
the classification data to only include the rows that correspond to subjects that
participants marked as "Yes, this should be included." There is an R script that
will help with this task included in the files:

- `filter_by_yes_include.R`

If you want to use this script, you will need an up-to-date R installation and the
powerful R package suite 'tidyverse'. We also strongly recommend RStudio, as it
offers some useful GUI options for working with R, including a handy environment window.

Once this is set up, gather this R file, the reduced question data file, and the
classification file into the same directory. Open the script in RStudio and perform the following steps:

1. Set the working directory to where your files are by selecting
**Session > Set Working Directory > To Source File Location**.
2. Adjust the filenames in the read_csv( ) statements to fit your filenames, if they differ
3. Adjust the output filename, if desired
4. Run all statements

This will create a new classifications file that only has the data we want. We
will want to use this classification data going forward.

### Text extraction & reduction (6)

There is one text extraction task in this project, corresponding to the page number
transcription. The process for text aggregation is similar to the one for the question
task, with one extra step. Since the configuration files have already been generated
in step 4, you will not need to run the configuration command again. Run the
extractor on the new, filtered classification data:

`panoptes_aggregation extract tar-pits-zooniverse-project-classifications-yes.csv
extractor_config_workflow_13300_V1.1.yaml`

(This extraction will overwrite the extraction files from step 4 if you are doing
it in the same directory - this is fine.)

If you try to reduce this data, it will complain because 1) there are probably
blank cells, since not all pages had page numbers, and 2) because (at the time of writing)
the reducer script reads the page numbers in as integer data. An OpenRefine transformation
can address these two issues. Open the "text_extractor_extractions" file in OpenRefine
and apply the following transformation:

```JSON
[
  {
    "op": "core/text-transform",
    "engineConfig": {
      "facets": [],
      "mode": "row-based"
    },
    "columnName": "data.text",
    "expression": "grel:if(value == null, 'No page', \"'\" + value + \"'\")",
    "onError": "keep-original",
    "repeat": false,
    "repeatCount": 10,
    "description": "Text transform on cells in column data.text using expression grel:if(value == null, 'No page #', \"'\" + value + \"'\")"
  }
]
```

This will fill the blank cells and add quotes to the numbers. You can now reduce the text data:

`panoptes_aggregation reduce text_extractor_extractions_transform.csv
Reducer_config_workflow_13300_V1.1_text_extractor.yaml`

### Dropdown menu extraction & reduction (7-8)

There are two dropdown tasks in this project. Each of the dropdown tasks is a "combo task"
which creates nested JSON data, and there is a python program provided by Zooniverse to de-nest
this data. Look for the following file in the repo:

- `remove_combo_nesting.py`

This script has three required flags, for input file, combo task, and output filename.
An example run of the script might look like this, assuming that T0 is one of the combo tasks:

`python remove_combo_nesting.py --classification_export tar-pits-zooniverse-project-classifications-yes.csv
 --combo_task_id T0 --output_file tar-pits-zooniverse-project-classifications-denested.csv`

Once you have run this twice (once for each combo task, T0 and T7), the extraction
and reduction process is similar to before:

`panoptes_aggregation extract tar-pits-zooniverse-project-classifications-denest2.csv
extractor_config_workflow_13300_V1.1.yaml`

`panoptes_aggregation reduce dropdown_extractor_extractions.csv
Reducer_config_workflow_13300_V1.1_dropdown_extractor.yaml`

Check the reduced dropdown data file -- there should be seven rows per subject,
corresponding to the seven dropdown tasks (four for paleontologist names, one each for day/month/year).

### Redacting the images (9-11)

The image redaction process involves using the shape data from the drawn boxes to alter images.

#### Rectangle extraction & reduction (9)

The shape (or rectangle) aggregation process is very similar to the other aggregation
steps. Either run the extraction step again, or use the 'shape_extractor_rectangle_extractions'
file from either steps 6 or 8, then reduce the shape data:

`panoptes_aggregation reduce shape_extractor_rectangle_extractions.csv
Reducer_config_workflow_13300_V1.1_shape_extractor_rectangle.yaml`

#### Preparing the redaction file (10)

Before we can continue with the redaction process, there are a few file preparation steps.
The first involves digging some image metadata out of JSON in the subjects .csv.
Open up the subjects file in OpenRefine. First, facet the workflow_id column
(text may be easier than number) and include only workflow ID 13300.
Then apply the following operations:

```JSON
[
  {
    "op": "core/column-addition",
    "engineConfig": {
      "facets": [
        {
          "type": "list",
          "name": "workflow_id",
          "expression": "value",
          "columnName": "workflow_id",
          "invert": false,
          "omitBlank": false,
          "omitError": false,
          "selection": [
            {
              "v": {
                "v": "13300",
                "l": "13300"
              }
            }
          ],
          "selectBlank": false,
          "selectError": false
        }
      ],
      "mode": "row-based"
    },
    "baseColumnName": "metadata",
    "expression": "grel:parseJson(value).get(\"id\")",
    "onError": "set-to-blank",
    "newColumnName": "img_filename",
    "columnInsertIndex": 5,
    "description": "Create column img_filename at index 5 based on column metadata using expression grel:parseJson(value).get(\"id\")"
  },
  {
    "op": "core/column-addition",
    "engineConfig": {
      "facets": [
        {
          "type": "list",
          "name": "workflow_id",
          "expression": "value",
          "columnName": "workflow_id",
          "invert": false,
          "omitBlank": false,
          "omitError": false,
          "selection": [
            {
              "v": {
                "v": "13300",
                "l": "13300"
              }
            }
          ],
          "selectBlank": false,
          "selectError": false
        }
      ],
      "mode": "row-based"
    },
    "baseColumnName": "metadata",
    "expression": "grel:parseJson(value).get(\"url\")",
    "onError": "set-to-blank",
    "newColumnName": "img_url",
    "columnInsertIndex": 5,
    "description": "Create column img_url at index 5 based on column metadata using expression grel:parseJson(value).get(\"url\")"
  }
]
```

Save the new .csv with a name you'll recognize as the transformed one.
Next, look for the following file in the repo:

- `prepare_for_redaction.R`

Open the file in RStudio. Set the working directory, and adjust the filenames as needed.
This script will prepare a file formatted specifically to be input into the
redaction program, so please at least leave the select( ) column order the same.
The file this script outputs will be the input for the next step.

#### Running the redaction program (11)

Start by finding the following file in the repo:

- `redact_v2.py`

I recommend putting this file and the file from the end of the previous step into
a separate directory, since the redaction program will be saving numerous
altered images to whichever directory it is run in. Next, run redact_v2.py from
the command line. The program expects the name of the input file as an argument, like so:

`python redact_v2.py --input_file [filename].csv`

If all is well, it should take a moment or so, automatically scrape the images from s3,
do the appropriate redactions, and save the image in the directory with the same filename.
The filename will match the filename in the modified subjects file from step 10.
The program is set to use some very light .jpg compression by default;
if you wish to turn off compression entirely, go into the script and
set `quality = 100` on the image save statements.

### Making the combined .csv (12)

At this point, between the reduced text and dropdown data and the redacted images,
(I think) you should technically have what you need. However, if you would like
the data in a form where it will be easier to check the classification data and
the image data against one another, there are a few further processing steps.
Find the following file in the repo:

- `combine_reduced_csvs.R`

Open the file in RStudio, set the working directory, and adjust the filenames.
This script 1) 'widens' the dropdown data so that the seven tasks are on the same
row for the same subject, and 2) joins the subject, text, and dropdown data.
There is also an operation to trim unnecessary columns, and rename columns if
necessary. The ones already listed in the script are only suggestions. The file
this script writes out should be a useful one to have around, and we can do some
further cleaning in the final step.

### Cleaning the combined .csv (13)

Taking a look at the data, you may have seen some cells that have data that looks
like [{'28c8443679985': 1}]. This is not ideal if this data is to be used for
any further analysis. The easiest way to extract these numbers and codes may be
doing a column transform in OpenRefine:

**Column dropdown > Edit cells > Transform... > substring(value, 3, -6)**

Lastly, find the file that has the 'translation' for each code in the data:

- `tarpits_codebook.txt`

and do find & replace operations on the relevant columns for month names and
paleontologist names:

**Column dropdown > Edit cells > Replace**

#### End Notes

This document written by Michael Lenard w/assistance from Justin Schell,
Max Ansorge, and Amber Ma.
If anything here is confusing, unclear, or in error, contact me at mclenard@umich.edu.
