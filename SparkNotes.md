- https://github.com/hhbyyh/DataFrameCheatSheet
- 

# spark and python

## PySpark notebook

```
export SPARK_HOME=/usr/local/spark-2.2.1-bin-hadoop2.7
export PATH=$PATH:$SPARK_HOME/bin
export PYTHONPATH=$PYTHONPATH:$SPARK_HOME/python:$SPARK_HOME/python/py4j-src.zip
# TO RUN PYSPARK as a NOTEBOOK--probably not what you want
PYSPARK_PYTHON=python PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS="notebook" pyspark --master='local[*]' --packages 
# TO RUN JUPYTER with PYTHON and pyspark on PYTHONPATH, do the above and then:
jupyter lab --ip='*' --no-browser
```

## anaconda python for pyspark

```
wget --quiet https://repo.continuum.io/archive/Anaconda3-5.1.0-Linux-x86_64.sh -O ~/anaconda.sh \
    && /bin/bash ~/anaconda.sh -b -p $HOME/conda

echo -e '\nexport PATH=$HOME/conda/bin:$PATH' >> $HOME/.bashrc && source $HOME/.bashrc

# install packages
conda install -y -c conda-forge jupyterlab
```


Add to **BOTH** .bashrc AND /etc/spark/conf/spark-env.sh

```
export PYSPARK_PYTHON=/home/hadoop/conda/bin/python
export PYSPARK_DRIVER_PYTHON=/home/hadoop/conda/bin/python
```


# Packages

- XML: --packages com.databricks:spark-xml_2.11:0.4.1


# EMR

## Job submission

- Python example. Notice how `--deploy-mode,cluster` is used. The path to the python file must be given in s3. Local files on the disk do not work in this mode.

```
aws emr --profile=emr --region=us-east-1 add-steps \
  --cluster-id j-1CGC1FEMNER0L \
  --steps Type=Spark,Name="Spark Program",ActionOnFailure=CONTINUE,Args=[--deploy-mode,cluster,s3://omics_metadata/xml_loader.py,s3n://omics_metadata/sra/reports/Mirroring/NCBI_SRA_Mirroring_20180201_Full/meta_study_set.xml.gz,s3n://omics_metadata/testing/study_parquet/,STUDY,-p,200]
```

- boto3

```
class Example:
  def run(self):
    session = boto3.Session(profile_name='emr-profile')
    client = session.client('emr')
    response = client.add_job_flow_steps(
    JobFlowId=cluster_id,
    Steps=[
        {
            'Name': 'string',
            'ActionOnFailure': 'CONTINUE',
            'HadoopJarStep': {
                'Jar': 'command-runner.jar',
                'Args': [
                    '/usr/bin/spark-submit',
                    '--verbose',
                    '--class',
                    'my.spark.job',
                    '--jars', '\'<coma, separated, dependencies>\'',
                    '<my spark job>.jar'
                ]
            }
        },
    ]
)
```

## Configuration

### Spark

- /etc/spark/conf/spark-defaults.conf

see: https://stackoverflow.com/questions/34003759/spark-emr-using-amazons-maximizeresourceallocation-setting-does-not-use-all

```
spark.maximizeResourceAllocation true # Otherwise, spark "downsizes" based on job
spark.default.parallelism        200  # Set default parallelism, since maximizeResourceAllocation does this ONLY AT START. 
```
Add packages via:

```
spark.jars.packages              com.databricks:spark-xml_2.11:0.4.1
```
All of this can be configured at cluster creation time by specifying: `classification=spark-defaults,properties=[spark.jars.packages=com.databricks:spark-xml_2.11:0.4.1]`, for example.

# DataFrames

## Schema work

The `df.dtypes` approach seems to be the most pythonic and useful. However, the way the thing below is written, we end up parsing strings! But it does work.

```
flat_cols = [c[0] for c in nested_df.dtypes if c[1][:6] != 'struct']
nested_cols = [c[0] for c in nested_df.dtypes if c[1][:6] == 'struct']
list([c + ".*" for c in nested_cols])
```

## Spark XML

```
import org.apache.spark.sql.SQLContext
import com.databricks.spark.xml._

val sqlContext = new SQLContext(sc)
val experiment = sqlContext.read
  .format("com.databricks.spark.xml")
  .option("rowTag", "EXPERIMENT")
  .load("s3n://omics_metadata/sra/reports/Mirroring/NCBI_SRA_Mirroring_20171213_Full/meta_experiment_set.xml.gz")
```

```
experiment.printSchema
```

```
root
 |-- DESIGN: struct (nullable = true)
 |    |-- DESIGN_DESCRIPTION: string (nullable = true)
 |    |-- LIBRARY_DESCRIPTOR: struct (nullable = true)
 |    |    |-- LIBRARY_CONSTRUCTION_PROTOCOL: string (nullable = true)
 |    |    |-- LIBRARY_LAYOUT: struct (nullable = true)
 |    |    |    |-- PAIRED: struct (nullable = true)
 |    |    |    |    |-- _NOMINAL_LENGTH: long (nullable = true)
 |    |    |    |    |-- _NOMINAL_SDEV: double (nullable = true)
 |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |-- SINGLE: string (nullable = true)
 |    |    |-- LIBRARY_NAME: string (nullable = true)
 |    |    |-- LIBRARY_SELECTION: string (nullable = true)
 |    |    |-- LIBRARY_SOURCE: string (nullable = true)
 |    |    |-- LIBRARY_STRATEGY: string (nullable = true)
 |    |    |-- TARGETED_LOCI: struct (nullable = true)
 |    |    |    |-- LOCUS: array (nullable = true)
 |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |-- PROBE_SET: struct (nullable = true)
 |    |    |    |    |    |    |-- DB: string (nullable = true)
 |    |    |    |    |    |    |-- ID: string (nullable = true)
 |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |-- _description: string (nullable = true)
 |    |    |    |    |    |-- _locus_name: string (nullable = true)
 |    |-- SAMPLE_DESCRIPTOR: struct (nullable = true)
 |    |    |-- IDENTIFIERS: struct (nullable = true)
 |    |    |    |-- EXTERNAL_ID: array (nullable = true)
 |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |-- _label: string (nullable = true)
 |    |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |    |-- PRIMARY_ID: string (nullable = true)
 |    |    |    |-- SECONDARY_ID: array (nullable = true)
 |    |    |    |    |-- element: string (containsNull = true)
 |    |    |    |-- SUBMITTER_ID: array (nullable = true)
 |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |    |-- UUID: string (nullable = true)
 |    |    |-- POOL: struct (nullable = true)
 |    |    |    |-- DEFAULT_MEMBER: struct (nullable = true)
 |    |    |    |    |-- IDENTIFIERS: struct (nullable = true)
 |    |    |    |    |    |-- EXTERNAL_ID: array (nullable = true)
 |    |    |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |    |    |    |-- PRIMARY_ID: string (nullable = true)
 |    |    |    |    |    |-- SUBMITTER_ID: struct (nullable = true)
 |    |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |    |-- MEMBER: array (nullable = true)
 |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |-- IDENTIFIERS: struct (nullable = true)
 |    |    |    |    |    |    |-- EXTERNAL_ID: array (nullable = true)
 |    |    |    |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |    |    |    |-- _label: string (nullable = true)
 |    |    |    |    |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |    |    |    |    |-- PRIMARY_ID: string (nullable = true)
 |    |    |    |    |    |    |-- SUBMITTER_ID: struct (nullable = true)
 |    |    |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |    |    |    |-- READ_LABEL: array (nullable = true)
 |    |    |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |    |    |-- _read_group_tag: string (nullable = true)
 |    |    |    |    |    |-- _accession: string (nullable = true)
 |    |    |    |    |    |-- _member_name: string (nullable = true)
 |    |    |    |    |    |-- _proportion: double (nullable = true)
 |    |    |    |    |    |-- _refcenter: string (nullable = true)
 |    |    |    |    |    |-- _refname: string (nullable = true)
 |    |    |-- _VALUE: string (nullable = true)
 |    |    |-- _accession: string (nullable = true)
 |    |    |-- _refcenter: string (nullable = true)
 |    |    |-- _refname: string (nullable = true)
 |    |-- SPOT_DESCRIPTOR: struct (nullable = true)
 |    |    |-- SPOT_DECODE_SPEC: struct (nullable = true)
 |    |    |    |-- READ_SPEC: array (nullable = true)
 |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |-- BASE_COORD: long (nullable = true)
 |    |    |    |    |    |-- EXPECTED_BASECALL_TABLE: struct (nullable = true)
 |    |    |    |    |    |    |-- BASECALL: array (nullable = true)
 |    |    |    |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |    |    |    |-- _match_edge: string (nullable = true)
 |    |    |    |    |    |    |    |    |-- _max_mismatch: long (nullable = true)
 |    |    |    |    |    |    |    |    |-- _min_match: long (nullable = true)
 |    |    |    |    |    |    |    |    |-- _read_group_tag: string (nullable = true)
 |    |    |    |    |    |    |-- _base_coord: long (nullable = true)
 |    |    |    |    |    |    |-- _default_length: long (nullable = true)
 |    |    |    |    |    |-- READ_CLASS: string (nullable = true)
 |    |    |    |    |    |-- READ_INDEX: long (nullable = true)
 |    |    |    |    |    |-- READ_LABEL: string (nullable = true)
 |    |    |    |    |    |-- READ_TYPE: string (nullable = true)
 |    |    |    |    |    |-- RELATIVE_ORDER: struct (nullable = true)
 |    |    |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |    |    |-- _follows_read_index: long (nullable = true)
 |    |    |    |    |    |    |-- _precedes_read_index: long (nullable = true)
 |    |    |    |-- SPOT_LENGTH: long (nullable = true)
 |    |-- _com: string (nullable = true)
 |-- EXPERIMENT_ATTRIBUTES: struct (nullable = true)
 |    |-- EXPERIMENT_ATTRIBUTE: array (nullable = true)
 |    |    |-- element: struct (containsNull = true)
 |    |    |    |-- TAG: string (nullable = true)
 |    |    |    |-- UNITS: string (nullable = true)
 |    |    |    |-- VALUE: string (nullable = true)
 |    |-- _com: string (nullable = true)
 |-- EXPERIMENT_LINKS: struct (nullable = true)
 |    |-- EXPERIMENT_LINK: array (nullable = true)
 |    |    |-- element: struct (containsNull = true)
 |    |    |    |-- URL_LINK: struct (nullable = true)
 |    |    |    |    |-- LABEL: string (nullable = true)
 |    |    |    |    |-- URL: string (nullable = true)
 |    |    |    |-- XREF_LINK: struct (nullable = true)
 |    |    |    |    |-- DB: string (nullable = true)
 |    |    |    |    |-- ID: string (nullable = true)
 |    |    |    |    |-- LABEL: string (nullable = true)
 |-- IDENTIFIERS: struct (nullable = true)
 |    |-- EXTERNAL_ID: struct (nullable = true)
 |    |    |-- _VALUE: string (nullable = true)
 |    |    |-- _namespace: string (nullable = true)
 |    |-- PRIMARY_ID: string (nullable = true)
 |    |-- SECONDARY_ID: array (nullable = true)
 |    |    |-- element: string (containsNull = true)
 |    |-- SUBMITTER_ID: array (nullable = true)
 |    |    |-- element: struct (containsNull = true)
 |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |-- _label: long (nullable = true)
 |    |    |    |-- _namespace: string (nullable = true)
 |    |-- UUID: string (nullable = true)
 |-- PLATFORM: struct (nullable = true)
 |    |-- ABI_SOLID: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |-- BGISEQ: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |-- CAPILLARY: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |-- COMPLETE_GENOMICS: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |-- HELICOS: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |-- ILLUMINA: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |    |-- _com: string (nullable = true)
 |    |-- ION_TORRENT: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |    |-- _com: string (nullable = true)
 |    |-- LS454: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |-- OXFORD_NANOPORE: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |    |-- PACBIO_SMRT: struct (nullable = true)
 |    |    |-- INSTRUMENT_MODEL: string (nullable = true)
 |-- PROCESSING: string (nullable = true)
 |-- STUDY_REF: struct (nullable = true)
 |    |-- IDENTIFIERS: struct (nullable = true)
 |    |    |-- EXTERNAL_ID: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |-- _label: string (nullable = true)
 |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |-- PRIMARY_ID: string (nullable = true)
 |    |    |-- SECONDARY_ID: string (nullable = true)
 |    |    |-- SUBMITTER_ID: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- _VALUE: string (nullable = true)
 |    |    |    |    |-- _label: string (nullable = true)
 |    |    |    |    |-- _namespace: string (nullable = true)
 |    |    |-- UUID: string (nullable = true)
 |    |-- _VALUE: string (nullable = true)
 |    |-- _accession: string (nullable = true)
 |    |-- _refcenter: string (nullable = true)
 |    |-- _refname: string (nullable = true)
 |-- TITLE: string (nullable = true)
 |-- _accession: string (nullable = true)
 |-- _alias: string (nullable = true)
 |-- _broker_name: string (nullable = true)
 |-- _center_name: string (nullable = true)
 |-- _com: string (nullable = true)
 |-- _ns1: string (nullable = true)
 |-- _ns2: string (nullable = true)
 |-- _xmlns: string (nullable = true)
 |-- _xsi: string (nullable = true)
 ```
 
## CSV files

see https://github.com/databricks/spark-csv#scala-api

```
val sqlContext = new SQLContext(sc)
val df = sqlContext.read
    .format("com.databricks.spark.csv")
    .option("header", "true") // Use first line of all files as header
    .option("inferSchema", "true") // Automatically infer data types
    .load("cars.csv")

val selectedData = df.select("year", "model")
selectedData.write
    .format("com.databricks.spark.csv")
    .option("header", "true")
    .save("newcars.csv")
```
