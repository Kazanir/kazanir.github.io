---
title: Drupal 8 Performance Profiling
layout: page
permalink: /profiling/
---

Since there is still some debate about Drupal 8's performance in the [performance-related meta issue on Drupal.org](https://www.drupal.org/node/1744302), I thought I would take a look at current performance numbers on several platforms and see how D8 is shaping up. (Since the time of this writing this issue has been close in favor of [this profiling-related meta](https://www.drupal.org/node/2470679) but most of the debate happened in the earlier thread.) Authenticated user performance and scaling of uncached page requests has never been one of Drupal's particularly strong points, and I was interested in gathering enough information to get an accurate picture of how things look right now. (As discussed at the end of the post, I also wanted to do it in a way that others could easily replicate.)

I hope the following data is useful.

#### Targets and Results ####

I profiled three targets: Drupal 7 with no page cache, Drupal 8 with no page cache, and Drupal 8 with the page cache. Data was populated by the `devel_generate` module as described in the README for the OSS Performance repository. More detail on how I assembled the Drupal 8 repository is included in its own target folder's README. The database dump, static file assets, and settings files used are all included in the target folder for all three targets. (Obviously I'm aware that D7 core and D8 core are far from a 1:1 comparison for a dozen different reasons and that's not my intent -- D7 stats are included as useful but not-all-determining background. More on this later.) All of the non-page-cached targets should be considered to have completely warm internal cache bins due to the number of warmup requests taken on each URL before the actual benchmarking run.

Each target was tested with a concurrency of 1, 5 and 20 on the following 4 PHP runtimes:

- PHP 5.6
- PHP 7.0 (nightly build from 24 May)
- HHVM 3.7.1
- HHVM 3.7.1 w/ Repo Authoritative mode

HHVM's "repo authoritative" mode is similar to PHP's `apc.stat = 0` in that it does not allow for changes to code files after the server has been booted. However, the restriction is actually more serious -- new code files cannot even be *added* after the server is booted (as the codebase is compiled into memory at boot time) and `eval()` and `create_function()` cannot be used. This means that D8 required a little sorcery to pre-compile all of Core's Twig templates. To do this (with many thanks to [Fabianx](https://www.drupal.org/u/fabianx) for his advice) I used a Drush script and fed it all the Twig templates in the repository, rendering them each once via Drush before compiling the HHVM codebase. (This setup script is in the profiling toolkit repo discussed later.)

The stats from the benchmarking tool are [available here](http://tiny.cc/d8perfstats). The raw JSON output is available for all targets on the second tab of the sheet. Here are the top-line results:

| Concurrency: 1          | Drupal 7, No Cache   | Drupal 8, No Cache   | Drupal 8, Page Cache |
|-------------------------|----------------------|----------------------|----------------------|
| PHP 5.6                 | `18.881431767338 ms` | `59.322388059701 ms` | `4.3355761143817 ms` |
| PHP 7 Nightly           | `9.6919250538884 ms` | `29.00537109375 ms`  | `2.4721870477577 ms` |
| HHVM 3.7.1              | `9.8024794772995 ms` | `33.148272017838 ms` | `2.7152219977343 ms` |
| HHVM 3.7.1 w/ Repo.Auth | `9.352187149495 ms`  | `24.877097315436 ms` | `1.9751959189708 ms` |

| Concurrency: 5          | Drupal 7, No Cache   | Drupal 8, No Cache   | Drupal 8, Page Cache |
|-------------------------|----------------------|----------------------|----------------------|
| PHP 5.6                 | `38.900183390097 ms` | `120.97404703974 ms` | `9.5226556343559 ms` |
| PHP 7 Nightly           | `19.206674053313 ms` | `59.194753577106 ms` | `5.1739456043208 ms` |
| HHVM 3.7.1              | `18.939017359553 ms` | `63.98563464837 ms`  | `5.7813411673458 ms` |
| HHVM 3.7.1 w/ Repo.Auth | `17.91012121212 ms`  | `47.262248295544 ms` | `3.6359491754163 ms` |

| Concurrency: 20         | Drupal 7, No Cache   | Drupal 8, No Cache   | Drupal 8, Page Cache |
|-------------------------|----------------------|----------------------|----------------------|
| PHP 5.6                 | `163.54299384826 ms` | `496.97964270877 ms` | `37.044550591612 ms` |
| PHP 7 Nightly           | `74.770701435466 ms` | `234.28212674122 ms` | `19.385923792941 ms` |
| HHVM 3.7.1              | `73.781373517787 ms` | `262.5686704695 ms`  | `21.350067339409 ms` |
| HHVM 3.7.1 w/ Repo.Auth | `69.725548552755 ms` | `186.66401497426 ms` | `10.07130574152 ms`  |

#### Profiling Data ####

In addition, I took performance profiles of most of the Drupal 8 targets for detailed analysis. [Blackfire.io](https://blackfire.io) was used to profile Drupal 8 on for PHP 5.6.9, with 200 requests for each of the following targets

- [Drupal 8, Warm Caches, Single Node](https://blackfire.io/profiles/57b82af3-402c-448f-9c90-2a65640872f6/graph)
- [Drupal 8, Warm Caches, Front Page](https://blackfire.io/profiles/d2d4ff59-9262-4578-8a00-96dcb3af128b/graph)
- [Drupal 8, Warm Caches, User Login](https://blackfire.io/profiles/f7a96bf9-85b5-401e-9a93-b0237f9e505c/graph)
- [Drupal 8, Page Cache, Front Page](https://blackfire.io/profiles/705a6e1f-3ec4-4c06-84f4-f208b76215f5/graph)

I also used the Linux tools perf utility to profile HHVM during its 6 runs, recording a callgraph which profiles the requests and function calls. This was turned into a flamegraph SVG using [Brendan Gregg's wonderful flamegraph utility](https://github.com/brendangregg/FlameGraph). These flamegraphs are really neat and can be found here:

- [Drupal 8, No Cache](/files/d8-nocache-flamegraph.svg)
- [Drupal 8, No Cache, Repo Auth](/files/d8-nocache-repo-flamegraph.svg)
- [Drupal 8, Page Cache](/files/d8-pagecache-flamegraph.svg)
- [Drupal 8, Page Cache, Repo Auth](/files/d8-pagecache-repo-flamegraph.svg)

#### Check My Work ####

The problem with performance profiling is that, in short, statistics are easy to get wrong. Unfortunately, a nice chart of stats is very readable and very convincing to the human mind in spite of how easy it is to accidentally end up with misleading stats. With that in mind, I have tried to document how I got here and make this process repeatable for anyone who wants to try the same stuff I did.

These tests were all run on an AWS `c4.xlarge` instance class. (I picked this because it roughly corresponds to the largest available Heroku Dyno size.) All of the tests had very repeatable results -- I didn't see any "noisy neighbor" effect from being on AWS and I used many different spot instances over the course of assembling this into a repeatable process.

The test itself warms up an entire fresh Drupal codebase (I used Beta 11 released on May 27th) along with a database dump and settings files from the OSS repo (mentioned below.) It first boots up the server and does 300 warmup requests. (There are several reasons for this, both warming up Drupal's internal caches and warming up the HHVM JIT compiler so it knows which code paths to optimize.) Then the test is run with a set concurrency for 2 minutes using Siege. (Siege 2.7.0 was used due to various bugs in the newer versions.) The default concurrency in Facebook's profiling kit is 200, but I reduced it to 1, 5 and 20 for these tests. Even at 20 it was evident that our instance was basically CPU-blocked. (The test machine has 4 logical cores; each AWS VCPU corresponds to a single hyperthread.)

I used a shell script of my own to provision the EC2 instance. I originally planned for this repo ([found here](https://www.github.com/Kazanir/maat)) to do the majority of the work, but midway into working on the provisioning scripts, I found that Facebook had taken care of a lot of the later legwork already. I quickly switched to using their OSS Performance toolkit. This means that my provisioner contains a lot of scaffolding for a different plan which can be safely ignored. This didn't affect the Facebook toolkit and the install/provisioning process in the README should work out of the box -- the most recent copy of Ma'at's provisioning script was used to take these benchmarks with no manual fixes or changes required.

I used Facebook's [OSS Performance toolkit](https://www.github.com/hhvm/oss-performance) to do the actual testing. This should be as simple as cloning this repository, tweaking your settings, and running the appropriate commands. Additional documentation for this toolkit [can be found here](https://github.com/facebook/hhvm/wiki/Profiling#strobelight). I created the pull request to add Drupal 8 as a target (which is what I used to run these benchmarks) and it is currently [under review here](https://github.com/hhvm/oss-performance/pull/43); at this point I expect only minor tweaks before it is merged.

My hope is that with all of the necessary plumbing documented in these repositories that these results should be fairly repeatable on similar hardware. In addition, fixes are quite obviously welcome -- if anyone sees trouble with the Drupal 8 target setup or other configuration bits that are affecting the profiling results then I'm happy to re-evaluate and re-run the tests. I hope that this transparency helps give the data more credibility than if it were a random pile of screenshots with no background information.

#### Takeaways ####

I think the main takeaway is that there is plenty to do to improve D8's performance yet, particularly for those sites which can't realistically make use of the page cache. I'm certainly a little biased here -- I work for Commerce Guys and have used Drupal to build stuff like support ticketing systems and community forum-type sites as well as e-commerce sites. And I think being able to build sites of that complexity is something that plays to Drupal's strengths as a content management framework.

That said, we've gained many features in the move to Drupal 8 and it isn't valid to try to make D7 to D8 a one-to-one comparison. In addition, given the performance increases available through new PHP runtimes, people in charge of serious, performance-hungry Drupal 8 deployments are hopefully still in good shape.

Many thanks to the HHVM team for their work on the benchmarking toolkit and for their help putting the Drupal 8 target together, as well as to advance help from `#drupal-contribute` with feedback on the draft version of this post. (Especially [Berdir](https://www.drupal.org/u/berdir) who prodded me until I figured out what was wrong with the initial PHP5 benchmarks...)

I'm keen to repeat these tests regularly and publish the data now that the process is fairly systematized and almost automatable. As noted above, feedback is extremely welcome (and the value of this information will also depend on figuring out if I have done anything wrong.) 
