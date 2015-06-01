---
title: Drupal 8 Performance Profiling
layout: page
permalink: /profiling/
---

After checking out the contents of [[meta] Resolve known performance regressions in Drupal 8](https://www.drupal.org/node/1744302) it became clear that we needed some serious numbers. Authenticated user performance and scaling of uncached page requests has never been one of Drupal's particularly strong suits, and after seeing some scary rumours I wanted to try to get a handle on things myself. Its easy for one set of half-baked statistics or another to spread and enter the public consciousness even though that information might not be reliable, so I have tried to generate a lot of numbers and document how I got to them so that the process is repeatable enough for anyone who wants to explore or verify this work.

I hope the following is useful.

#### Targets and Results ####

I profiled three targets: Drupal 7 with no page cache, Drupal 8 with no page cache, and Drupal 8 with the page cache. Data was populated by the `devel_generate` module as described in the README for the OSS Performance repository. More detail on how I assembled the Drupal 8 repository is included in its own target folder's README. The database dump, static file assets, and settings files used are all included in the target folder for all three targets. (Obviously I'm aware that D7 core and D8 core are far from a 1:1 comparison for a dozen different reasons and that's not my intent -- D7 stats are included as useful but not-all-determining background comparison.)

Each target was tested with a concurrency of 1 and 20 on the following 4 PHP runtimes:

- PHP 5.6
- PHP 7.0 (nightly build from 24 May)
- HHVM 3.7.1
- HHVM 3.7.1 w/ Repo Authoritative mode

HHVM's "repo authoritative" mode is similar to PHP's `apc.stat = 0` in that it does not allow for changes to code files after the server has been booted. However, the restriction is actually more serious -- new code files cannot even be *added* after the server is booted (as the codebase is compiled into memory at boot time) and `eval()` and `create_function()` cannot be used. This means that D8 required a little sorcery to pre-compile all of Core's Twig templates. To do this (with many thanks to [Fabianx](https://www.drupal.org/u/fabianx) for his advice) I used a Drush script and fed it all the Twig templates in the repository, rendering them each once via Drush before compiling the HHVM codebase. (This setup script is in the profiling toolkit repo discussed later.)

Here are the single-request results:


| Concurrency: 1          | Drupal 7             | Drupal 8, No Cache   | Drupal 8, Page Cache |
|-------------------------|----------------------|----------------------|----------------------|
| PHP 5.6                 | `75.267339218159 ms` |  `219.2673992674 ms` | `22.322763306908 ms` |
| PHP 7 Nightly           | ` 9.661418627776 ms` |  ` 37.9087428206 ms` | ` 2.001863401278 ms` |
| HHVM 3.7.1              | ` 9.580457863304 ms` |  ` 42.6652329749 ms` | ` 2.219886789312 ms` |
| HHVM 3.7.1 w/ Repo.Auth | ` 9.879979570990 ms` |  ` 32.3803814713 ms` | ` 1.642432380587 ms` |


The stats from the benchmarking tool (a thin layer on top of Siege) are [available here](http://tiny.cc/d8perfstats). The raw JSON output (with more stats) is available for all targets on the second tab of the sheet.

#### Profiling Data ####

In addition, I took performance profiles of most of the targets. [Blackfire.io](https://blackfire.io) was used to profile all targets for PHP5, with 500 requests for D7 and page cached D8, and 100 for D8 with no page cache. (Because it is slow.) The Blackfire profiles are public and can be viewed/downloaded here:

- [Drupal 7, No Cache](https://blackfire.io/profiles/b8de711a-20a7-4571-a8a7-c84f812b92b9/graph)
- [Drupal 8, No Cache](https://blackfire.io/profiles/41ac1132-2602-43f9-aec1-eb4f8ad20203/graph)
- [Drupal 8, Page Cache](https://blackfire.io/profiles/9b630106-eade-4f48-8ba1-a05e0c7d98cb/graph)

I also used the Linux tools perf utility to profile HHVM during its 6 runs, recording a callgraph which profiles the requests and function calls. [These files can be found here.](https://paddedhelmets.s3.amazonaws.com/d8perfstats/index.html)

Be warned: the contents of each of these zips are roughly 500 MB apiece. Once you have unzipped (`tar -xjf`) one of these files, you'll want to move the `.map` file to your `/tmp/` folder; this is where the perf tool looks for the function map files. Once you've done this, the data file's callgraph and profile info can viewed with perf, like so:

```
perf report -g -i d7-repoauth-perf-10552.data
```

#### Check My Work ####

The problem with performance profiling is that, in short, statistics are easy to get wrong. Unfortunately, a nice chart of stats is very readable and very convincing to the human mind in spite of how easy it is to accidentally end up with misleading stats. With that in mind, I have tried to document how I got here and make this process repeatable for anyone who wants to try the same stuff I did.

These tests were done on an AWS `c4.xlarge` instance class running the Ubuntu 14.04 AMI provided by Canonical. I originally picked this class because its RAM corresponds to the largest available Heroku Dyno type, the Performance (PX) Dyno, but it appears that the original PX Dynos were based on the `c1.xlarge` class (which had twice as many logical CPU cores.) It is not clear whether new PX Dynos are being provisioned as `c4.xlarge` or `c4.2xlarge`. Either way, the `c4.xlarge` has 4 logical cores and the results were fairly consistent -- minimal evidence of the "noisy neighbors" effect from virtualized hosting.

I used a shell script of my own to provision the EC2 instance. I originally planned for this repo ([found here](https://www.github.com/Kazanir/maat)) to do the majority of the work, but midway into working on the provisioning scripts and testing I found that Facebook had taken care of a lot of the later legwork already, so I switched midstream to using their OSS Performance toolkit. This means that Ma'at has a bunch of unfinished work around an Apache-wildcard based setup for multiple Drupal testing services, but this didn't affect the Facebook toolkit and the install/provisioning process in the README should work out of the box -- the most recent copy of Ma'at's provisioning script was used to take these benchmarks.

I used Facebook's [OSS Performance toolkit](https://www.github.com/hhvm/oss-performance) to do the actual testing. This should be as simple as cloning this repository, tweaking your settings, and running the appropriate commands. Additional documentation for this toolkit [can be found here](https://github.com/facebook/hhvm/wiki/Profiling#strobelight). I created the pull request to add Drupal 8 as a target (which is what I used to run these benchmarks) and it is currently [under review here](https://github.com/hhvm/oss-performance/pull/43); at this point I expect only minor tweaks before it is merged.

My hope is that with all of the necessary plumbing documented in these repositories that these results should be fairly repeatable on similar hardware. In addition, fixes are quite obviously welcome -- if anyone sees trouble with the Drupal 8 target setup or other configuration bits that are affecting the profiling results then I'm happy to re-evaluate and re-run the tests. I hope that this transparency helps give the data more credibility than if it were a random pile of screenshots with no background information.


