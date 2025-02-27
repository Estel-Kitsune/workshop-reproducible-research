In a typical bioinformatics project, considerable efforts are spent on tweaking
parameters for the various programs involved. It would be inconvenient if you
had to change in the shell scripts themselves every time you wanted to run with
a new setting. Luckily, there is a better option for this: the `params`
keyword.

```python
rule some_rule:
    output:
        "..."
    input:
        "..."
    params:
        cutoff=2.5
    shell:
        """
        some_program --cutoff {params.cutoff} {input} {output}
        """
```

Most of the programs are run with default settings in the MRSA workflow and 
don't use the `params:` directive. However, the `get_SRA_by_accession` rule 
is an exception. Here the remote address for each of the files to download 
is passed to the shell directive via:

```python
def get_sample_url(wildcards):
    samples = {
        "SRR935090": "https://figshare.scilifelab.se/ndownloader/files/39539767",
        "SRR935091": "https://figshare.scilifelab.se/ndownloader/files/39539770",
        "SRR935092": "https://figshare.scilifelab.se/ndownloader/files/39539773"
    }
    return samples[wildcards.sample_id]
    
rule get_SRA_by_accession:
    """
    Retrieve a single-read FASTQ file
    """
    output:
        "data/raw_internal/{sample_id}.fastq.gz"
    params:
        url = get_sample_url
    shell:
        """
        wget -O - {params.url} | seqtk sample - 25000 | gzip -c > {output[0]}
        """

```

You may recognize this from [page 2](snakemake-2-the-basics.md) of this 
tutorial where we used input functions to generate strings and lists of 
strings for the `input:` section of a rule. Using a function to return 
values based on the wildcards also works for `params:`. Here `sample_id` is 
a wildcard which in this specific workflow can be either `SRR935090`, 
`SRR935091`, or `SRR935092`. The wildcards object is passed to the function 
`get_sample_url` and depending on what output the rule is supposed to 
generate, `wildcards.sample_id` will take the value of either of the three 
sample ids. The `samples` variable defined in the function is a Python 
[dictionary](https://docs.python.org/3/tutorial/datastructures.html#dictionaries)
that has the URLs for each sample_id hardcoded. This dictionary is used to 
convert the value of the `sample_id` wildcard to a URL, which is returned by 
the function. Finally, in the `shell:` directive we access the `url` parameter 
with `{params.url}`. (We could have written three separate rules to download 
the samples, but it's easy to see how that can become impractical.)

Let's add another parameter to the `get_SRA_by_accession` rule. As you can 
see in the shell command the fastq file downloaded by `wget` gets piped 
directly (the `-O -` part means send contents to STDOUT) to the `seqtk sample` 
command which reads from STDIN and outputs 25000 randomly sampled reads (out 
of the 100,000 contained in the example fastq file). Change in the rule to 
use the parameter `max_reads` instead and set the value to 20000. If you 
need help, click to show the solution below.

<details>
<summary> Click to show </summary>


```python
rule get_SRA_by_accession:
    """
    Retrieve a single-read FASTQ file
    """
    output:
        "data/raw_internal/{sample_id}.fastq.gz"
    params:
        url = get_sample_url,
        max_reads = 20000
    shell:
        """
        wget -O - {params.url} | seqtk sample - {params.max_reads} | gzip -c > {output[0]}
        """
```

</details>

Now run through the workflow. Because there's been changes to the `get_SRA_by_accession`
rule this will trigger a re-run of the rule for all three accessions. In addition
all downstream rules that depend on output from `get_SRA_by_accession` are re-run. 

As you can see the parameter values we set in the `params` section don't have 
to be static, they can be any Python expression. In particular, Snakemake 
provides a global dictionary of configuration parameters called `config`. 
Let's modify `get_SRA_by_accession` to look something like this in order to 
make use of this dictionary:

```python
rule get_SRA_by_accession:
    """
    Retrieve a single-read FASTQ file
    """
    output:
        "data/raw_internal/{sample_id}.fastq.gz"
    params:
        url = get_sample_url,
        max_reads = config["max_reads"]
    shell:
        """
        wget -L {params.url} | seqtk sample - {params.max_reads} | gzip -c > {output[0]}
        """
```

Note that Snakemake now expects there to be a key named `max_reads` in the config 
dictionary. If we don't populate the dictionary somehow the dictionary will be 
empty so if you were to run the workflow now it would trigger a `KeyError` (try 
running `snakemake -s snakefile_mrsa.smk -n` to see for yourself). 
In order to populate the config dictionary with data for the workflow we could 
use the `snakemake --config KEY=VALUE` syntax directly from the command line 
(_e.g._ `snakemake --config max_reads=20000 -s snakefile_mrsa.smk`).
However, from a reproducibility perspective, it's not optimal to set parameters 
from the command line, since it's difficult to keep track of which parameter 
values that were used. 

A much better alternative is to use the `--configfile FILE` option to supply a 
configuration file to Snakemake. In this file we can collect all the 
project-specific settings, sample ids and so on. This also enables us to write 
the Snakefile in a more general manner so that it can be better reused between 
projects. Like several other files used in these tutorials, this file should be 
in [yaml format](https://en.wikipedia.org/wiki/YAML). Create the file below and 
save it as `config.yml`.

```yaml
max_reads: 25000
```

If we now run Snakemake with `--configfile config.yml`, it will parse this file
to form the `config` dictionary. If you want to overwrite a parameter value,
*e.g.* for testing, you can still use the `--config KEY=VALUE` flag, as in 
`--config max_reads=1000`.

> **Tip** <br>
> Rather than supplying the config file from the command line you could also
> add the line `configfile: "config.yml"` to the top of your Snakefile. Keep in 
> mind that with such a setup Snakemake will complain if the file `config.yml` 
> is not present.

> **Quick recap** <br>
> In this section we've learned:
>
> - How to set parameter values with the `params` directive.
> - How to run Snakemake with the `config` variable and with a configuration file.