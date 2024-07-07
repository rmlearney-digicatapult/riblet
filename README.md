# riblet

`riblet` is a command-line tool for efficiently diffing and syncing large [sets](https://en.wikipedia.org/wiki/Set_(mathematics)). Sets are stored in unordered, newline-delimited files, which are assumed to not contain duplicates.

The `riblet build` command performs a "riblet transform", which converts a set into a sequence of coded symbols. These symbols can be used to rapidly identify if two sets differ and, if so, what the differences actually are. Best of all, the symbols form an incremental error-correcting code, which means that you only need to download or process a number of them proportional to the amount of differences between the two sets: The sizes of the sets themselves do not matter.

Riblet is based on an algorithm called Rateless Invertible Bloom Lookup Tables (RIBLT), introduced in the paper [Practical Rateless Set Reconciliation](https://arxiv.org/pdf/2402.02668) by Lei Yang et al. For an approachable tutorial on this algorithm, see the following video series: [1](https://www.youtube.com/watch?v=eCUm4U3WDpM), [2](https://www.youtube.com/watch?v=BIN2a-CIvNA), [3](https://www.youtube.com/watch?v=B943F4IpLWo). Also see Dr. Yang's [presentation](https://www.youtube.com/watch?v=oS6bUx9KUGc).

<!-- TOC FOLLOWS -->
<!-- START OF TOC -->
<!-- DO NOT EDIT! Auto-generated by md-toc: https://github.com/hoytech/md-toc -->

* [Compilation](#compilation)
* [Usage](#usage)
    * [Building a `.riblet` File](#building-a-riblet-file)
        * [Memory Usage](#memory-usage)
        * [Duplicates](#duplicates)
    * [Dumping a `.riblet` File](#dumping-a-riblet-file)
    * [Diffing Two `.riblet` Files](#diffing-two-riblet-files)
        * [Streaming](#streaming)
        * [Symmetric Difference and Merging Files](#symmetric-difference-and-merging-files)
* [Delta Encoding](#delta-encoding)
* [Performance Notes](#performance-notes)
* [Security](#security)
* [Application: All Of Domains](#application-all-of-domains)
    * [Experiment](#experiment)
    * [Conclusion](#conclusion)
* [Author](#author)

<!-- END OF TOC -->

## Compilation

Riblet requires a C++20 compiler and OpenSSL:

    ## Debian/Ubuntu
    sudo apt install -y build-essential libssl-dev

    ## Redhat
    sudo yum install make g++ openssl-dev

Then in the repo directory, build the `riblet` binary with these commands:

    git submodule update --init
    make setup-golpe
    make -j

To run the test-suite, invoke the following:

    cd test
    perl test.pl


## Usage

### Building a `.riblet` File

The first step is usually to generate a `.riblet` file. Suppose you have a CSV file named `file.csv`. Run the `riblet build` command:

    riblet build --num 1000 file.csv

This will create a new file `file.csv.riblet` containing 1000 coded symbols. You need about ~1.35 symbols for each difference you expect, so this file should be able to decode about 740 differences (`1000 / 1.35`).

You can also choose a specific output file instead of having `.riblet` automatically appended to the input filename:

    riblet build --num 1000 file.csv custom-name.riblet

Or, you can output to stdout by using `-` as a filename:

    riblet build --num 1000 file.csv -

The input data can be taken from stdin too:

    riblet build --num 1000 - -

`riblet build` will not overwrite an output file by default. To force this, use the `--rebuild` flag.

#### Memory Usage

When you specify a number of coded symbols with `--num`, the memory usage is unaffected by the input size and is linear in the number of coded symbols.

If the output is specified as stdout (using `-`) then the `--num` flag is optional. If omitted, `riblet build` will output an infinite number of coded symbols. However, in this mode the memory usage *will* be proportional to the input size (plus the growing number of coded symbols, although we plan to fix that).

#### Duplicates

Input files must *not* have any duplicate lines. If you aren't sure about this, when running `riblet build` you can check for dups by passing the `--dup-check` flag (fails if it detects a dup) or `--dup-skip` (silently skips dups). Note that both of these flags require a memory overhead proportional to the input size (approximately 16 bytes per record).


### Dumping a `.riblet` File

Normally `.riblet` files are packed binary files, but you can see a human-readable dump of their contents with the `riblet dump` command. This is mostly useful for development and debugging:

    $ riblet dump file.csv.riblet
    RIBLT Header
      Build Timestamp = 1718996710
      Input Filename = "file.csv"
      Input Records = 1005
      Output Symbols = 1000
    -----------------------
    Symbol 0
      val = '160000005b6d5f2d2b772b2f6f0e59433157172f517e132a2808'
      hash = '45775d387be4a4be2a31d32b858e382d'
      count = 1005
    Symbol 1
      val = '160000004861156a4d7d4a56032e4e1838276c63354e5b6c147e'
      hash = 'e57c2e130b6761c373a9b299341d0a3d'
      count = 647
    ...

As with all `riblet` commands, `-` can be used as an input file to signify stdin.



### Diffing Two `.riblet` Files

Suppose you copied `file.csv` to `file2.csv` and deleted one line and added two new ones. After building riblet files for each, you can diff them like so:

    riblet diff file.csv.riblet file2.csv.riblet

The output will be something like this:

    +added another line
    -bad line
    +added line

Notice that the order of the differences is undefined.

Just like with the unix `diff -u` command, lines that start with `+` were added and lines starting with `-` were deleted. The cool thing about riblet is that while `diff` needs to read the entire contents of both files, `riblet diff` only needs to read about 1.35 times the number of *differences*. So, in this case, it would've read about `3 * 1.35 ~= 4` symbols from each `.riblet` file. **This is unaffected by the input file sizes**. `file.csv` and `file2.csv` could've been tens of lines long or millions: it would make no difference to the work required for this diff operation.

If either of the `.riblet` files are too short, `riblet diff` will fail with an error such as the following:

    riblet error: insufficient coded symbols from source A

#### Streaming

Because the amount of data needed from the `.riblet` files is small when data-sets are similar, and because it always streams data starting from the beginning of the file, `riblet diff` can be used to minimise bandwidth required when synchronising data-sets over the network.

For example, suppose a data maintainer updates a data-set periodically, and publishes a corresponding `.riblet` file on their website. You can figure out what has changed since you downloaded the file:

    riblet diff file.csv.riblet <(curl https://example.com/file.csv.riblet)

After downloading data proportional to the number of differences, `riblet` will close the file descriptor, which closes the HTTPS connection (the OS will kill the `curl` process with [SIGPIPE](https://www.pixelbeat.org/programming/sigpipe_handling.html)). If the files are similar, this saves bandwidth, memory, and CPU relative to downloading the entire file and `diff`ing it.

If desired, riblet files can be compressed, in which case you might use something like the following:

    riblet diff \
        file.csv.riblet \
        <(curl https://example.com/file.csv.riblet.gz | gzip -d)

Of course, any command can be used instead of `curl`. If CPU/memory is cheap but bandwidth is dear, the riblet files can even be generated on-demand:

    riblet diff \
        <(riblet build file.csv -) \
        <(ssh user@example.com riblet build file.csv -)

Notice how `--num` is not provided to either of the `riblet build`s. This means they will keep generating symbols "forever" until the difference can be computed. Although this is guaranteed to not run out of symbols, each side will have to allocate an amount of memory proportional to its set size. So, if you know an upper-bound on the number of differences, you may want to provide `--num` so as to limit peak memory consumption.

#### Symmetric Difference and Merging Files

`riblet diff` also takes a `--symmetric` flag. Instead of prefixing the detected differences with `+` and `-` symbols, it just prints them out with no prefix at all:

    $ riblet diff --symmetric file.csv.riblet file2.csv.riblet > file.diff
    $ cat file.diff
    added another line
    bad line
    added line

In mathematical terminology, this is called the [symmetric difference](https://en.wikipedia.org/wiki/Symmetric_difference) between the two input sets: the set of items that exist in only one set (but not both). Although it is in a sense throwing out some information (which set did each line come from?), it is useful for merging a set of differences with a source file. For example, if we have `file.csv` and `file.diff`, we can compute `file2.csv` with the following command:

    sort file.csv file.diff | uniq -u > file2.csv

The `-u` flag tells `uniq` to only output unique (non-duplicated) lines. If a line appears in both the original and the diff, it must be a deletion line, in which case `uniq -u` will *not* print it. On the other hand, if it only appears in one of the files, it must've been from either the source file or an addition line from the diff, so it *will* be printed.

`sort` uses an external sort algorithm, meaning it is possible to sort files too large to fit in memory (useful tip: check out the `-T` and `--compress-program` flags).

Although not required by `riblet`, if you are always careful to keep your data files sorted, you can optimise this significantly with the `-m` flag which tells `sort` it can simply merge its inputs. This allows `sort` to output records immediately, without having to read its whole inputs first. For example:

    sort -m file.csv <(sort file.diff) | uniq -u > file2.csv

It is not currently possible to incrementally update a `.riblet` file from a diff, but we plan to add this functionality. When we do, it will also be possible to merge (coalesce) diffs, thanks to the linearity of the RIBLT algorithm. We're also planning on adding an efficient way to append new records onto the source file for grow-only use-cases.


## Delta Encoding

Another approach to incrementally syncing files is [delta encoding](https://en.wikipedia.org/wiki/Delta_encoding) ([xdelta](https://en.wikipedia.org/wiki/Xdelta), [open-vcdiff](https://github.com/google/open-vcdiff/), etc). This stores the differences themselves in a file, which can be applied to a previously downloaded version. However, for delta encoding to work, you need to apply a delta to the exact same file that the other side generated it from. If you miss downloading a few deltas, you are either out-of-luck or perhaps you have to find multiple delta files to download and apply (in the correct order).

With riblet, the remote side does not need to store multiple delta files, and it doesn't matter if you're several periods behind. Furthermore, you can make edits to your own copy of the file without invalidating the sync, whereas if this is done with delta-encoding it would cause data corruption. In addition, while delta encoding relies on a single canonical publisher, riblet can sync files that are created and edited by multiple unrelated parties.

With riblet, each user can decide how many coded symbols to maintain/publish, and they do not need to coordinate this decision with anyone. This means that large sequences of coded symbols can be published, but clients can keep cheap/small sequences if they do not expect to need to sync many differences.


## Performance Notes

* When computing the coded symbols, the size of each symbol's value is the maximum of the input records used to build it. Since the first element is the combination of *all* records, it is the size of the largest element in the set. As the coded symbols are generated, since they incorporate fewer records on average, their size will decrease (again, on average). For this reason, riblet works best with sets of similarly-sized records, or at least ones without too many large outliers.
* Currently `riblet build` is CPU-bound, and runs on a single thread. This can easily be improved, thanks to the linearity of the RIBLT algorithm. Each thread could compute its own set of coded symbols over a sub-set of the input, and then have them combined into the final stream at output-time. Parallelising `riblet diff` is more difficult, since the "peel" algorithm isn't embarrassingly parallel in the same way.
* Although originally designed for it, at this time it is not clear to us if the RIBLT algorithm will work well for the online-server use-case. As a randomised data-structure, RIBLT will require considerable random access IO to maintain and/or generate, which may be in short supply on a busy server. RIBLT also cannot amortise any work when adjacent records are missing/present (an extremely common use-case), and it degrades poorly when tasked with common base-cases such as where one side has all/none/few of the records. In general, we think if the records to be synced have any temporal/spatial locality (for example, if they contain a timestamp), [Range-Based Set Reconciliation](https://logperiodic.com/rbsr.html) will be superior. More research/experimentation is needed to prove/disprove this.


## Security

Because the RIBLT algorithm combines hashes with XOR and compares these values to other hashes, it is trivially vulnerable to adversarial input, as discussed in [this github issue](https://github.com/yangl1996/riblt/issues/3). Fortunately, there is reason to believe actually exploiting this in real-world scenarios may be impractical (see the github issue).

Even if adversarial inputs do become problematic, RIBLT can likely be adapted to use a more secure incremental hash algorithm.




## Application: All Of Domains

In order to gather some concrete data about riblet, we performed an experiment with a non-trivial sized data-set: The set of all registered domain names.

### Experiment

We start with two files, `domains1.txt` and `domains2.txt`, which contain the data-sets on subsequent days. Each file contains ~266 million records and consumes ~4.6 GB of disk-space. They are sorted and look something like this:

    000000000000000000000000000000000000000000000000000000000000000.com
    000000000000000000000000000000000000000000000000000000000000000.co.uk
    000000000000000000000000000000000000000000000000000000000000000.limited
    ...
    zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz.ru
    zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz.vlaanderen
    zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz.zone

Running `riblet build` for each takes about 20 minutes (we ran them in parallel).

After building the `.riblet` files, computing the symmetric difference takes only 5 seconds:

    $ time riblet diff domains1.txt.riblet domains2.txt.riblet | sort > diff-riblet

    real    0m5.037s
    user    0m4.858s
    sys     0m0.180s

Whereas the same diffing operation using `sort -m` takes about 2 minutes:

    $ time sort -m domains1.txt domains2.txt | uniq -u > diff-sort

    real    1m50.701s
    user    2m13.256s
    sys     0m22.839s

The resulting `diff-riblet` and `diff-sort` files are identical, ~468 thousand lines of the following:

    0000500.com
    0000520.xyz
    0000cpz.com
    ...
    zzzxyz.top
    zzzzzu.com
    zzzzzv.com

Surprisingly, there were slightly more records removed than added, in violation of [Linus' Law](https://git-scm.com/docs/pack-heuristics).

Both of the above tests were performed with all files warm in the page cache. However, after evicting the files from the cache and re-running `riblet diff`, we can use [vmtouch](https://hoytech.com/vmtouch/) to confirm that only prefixes of the `.riblet` files were actually accessed:

    $ vmtouch -v *.riblet
    domains1.txt.riblet
    [OOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOo                    ] 11612/17671
    domains2.txt.riblet
    [OOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOo                    ] 11612/17672

               Files: 2
         Directories: 0
      Resident Pages: 23224/35343  90M/138M  65.7%

About 66% of each file was loaded, which is roughly what we'd expect: `468048 * 1.35 / 1e6 = 0.631865`. The extra ~2% is probably read-ahead combined with the fact that earlier coded symbols are larger.

In terms of file sizes, each `.riblet` file with 1 million coded symbols takes about 72 MB, about 48 MB of which were accessed for the diff. By comparison, the raw symmetric difference was about 8 MB.

### Conclusion

It's difficult to say whether riblet is a good fit for this application. It performed reliably and predictably, but the `.riblet` data needed for a day's difference is about 6x the size of the symmetric difference. This relative overhead will increase somewhat with compression, since sorted lists of domain names compress better than `.riblet` files (which are randomly ordered and contain lots of hash function outputs).

There are a few possible ways to reduce the overhead of `.riblet` files and we could probably cut it down about a third by truncating hashes and better packing some of the symbol meta-data, but much of this overhead is an unavoidable property of the RIBLT algorithm. Generating the `.riblet` files takes considerable time, but there is a lot of room to optimise this: parallelisation, faster hash functions, etc.

Given the volume of updates and a daily sync cadence, a delta encoding approach might be preferable for this application. However, if you needed to sync this list hourly (or more frequently) then being able to gracefully recover from missed delta updates might tilt the scales in favour of riblet.



## Author

(C) 2024 Doug Hoyte

Code and tests are MIT licensed.

Riblet is a [Log Periodic](https://logperiodic.com) project.
