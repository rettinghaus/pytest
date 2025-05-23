
Flaky tests
-----------

A "flaky" test is one that exhibits intermittent or sporadic failure, that seems to have non-deterministic behaviour. Sometimes it passes, sometimes it fails, and it's not clear why. This page discusses pytest features that can help and other general strategies for identifying, fixing or mitigating them.

Why flaky tests are a problem
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Flaky tests are particularly troublesome when a continuous integration (CI) server is being used, so that all tests must pass before a new code change can be merged. If the test result is not a reliable signal -- that a test failure means the code change broke the test -- developers can become mistrustful of the test results, which can lead to overlooking genuine failures. It is also a source of wasted time as developers must re-run test suites and investigate spurious failures.


Potential root causes
^^^^^^^^^^^^^^^^^^^^^

System state
~~~~~~~~~~~~

Broadly speaking, a flaky test indicates that the test relies on some system state that is not being appropriately controlled - the test environment is not sufficiently isolated. Higher level tests are more likely to be flaky as they rely on more state.

Flaky tests sometimes appear when a test suite is run in parallel (such as use of `pytest-xdist`_). This can indicate a test is reliant on test ordering.

-  Perhaps a different test is failing to clean up after itself and leaving behind data which causes the flaky test to fail.
- The flaky test is reliant on data from a previous test that doesn't clean up after itself, and in parallel runs that previous test is not always present
- Tests that modify global state typically cannot be run in parallel.


Overly strict assertion
~~~~~~~~~~~~~~~~~~~~~~~

Overly strict assertions can cause problems with floating point comparison as well as timing issues. :func:`pytest.approx` is useful here.

Thread safety
~~~~~~~~~~~~~

pytest is single-threaded, executing its tests always in the same thread, sequentially, never spawning any threads itself.

Even in case of plugins which run tests in parallel, for example `pytest-xdist`_, usually work by spawning multiple *processes* and running tests in batches, without using multiple threads.

It is of course possible (and common) for tests and fixtures to spawn threads themselves as part of their testing workflow (for example, a fixture that starts a server thread in the background, or a test which executes production code that spawns threads), but some care must be taken:

* Make sure to eventually wait on any spawned threads -- for example at the end of a test, or during the teardown of a fixture.
* Avoid using primitives provided by pytest (:func:`pytest.warns`, :func:`pytest.raises`, etc) from multiple threads, as they are not thread-safe.

If your test suite uses threads and your are seeing flaky test results, do not discount the possibility that the test is implicitly using global state in pytest itself.

Related features
^^^^^^^^^^^^^^^^

Xfail strict
~~~~~~~~~~~~

:ref:`pytest.mark.xfail ref` with ``strict=False`` can be used to mark a test so that its failure does not cause the whole build to break. This could be considered like a manual quarantine, and is rather dangerous to use permanently.


PYTEST_CURRENT_TEST
~~~~~~~~~~~~~~~~~~~

:envvar:`PYTEST_CURRENT_TEST` may be useful for figuring out "which test got stuck".
See :ref:`pytest current test env` for more details.


Plugins
~~~~~~~

Rerunning any failed tests can mitigate the negative effects of flaky tests by giving them additional chances to pass, so that the overall build does not fail. Several pytest plugins support this:

* `pytest-rerunfailures <https://github.com/pytest-dev/pytest-rerunfailures>`_
* `pytest-replay <https://github.com/ESSS/pytest-replay>`_: This plugin helps to reproduce locally crashes or flaky tests observed during CI runs.
* `pytest-flakefinder <https://github.com/dropbox/pytest-flakefinder>`_ - `blog post <https://blogs.dropbox.com/tech/2016/03/open-sourcing-pytest-tools/>`_

Plugins to deliberately randomize tests can help expose tests with state problems:

* `pytest-random-order <https://github.com/jbasko/pytest-random-order>`_
* `pytest-randomly <https://github.com/pytest-dev/pytest-randomly>`_


Other general strategies
^^^^^^^^^^^^^^^^^^^^^^^^

Split up test suites
~~~~~~~~~~~~~~~~~~~~

It can be common to split a single test suite into two, such as unit vs integration, and only use the unit test suite as a CI gate. This also helps keep build times manageable as high level tests tend to be slower. However, it means it does become possible for code that breaks the build to be merged, so extra vigilance is needed for monitoring the integration test results.


Video/screenshot on failure
~~~~~~~~~~~~~~~~~~~~~~~~~~~

For UI tests these are important for understanding what the state of the UI was when the test failed. pytest-splinter can be used with plugins like pytest-bdd and can `save a screenshot on test failure <https://pytest-splinter.readthedocs.io/en/latest/#automatic-screenshots-on-test-failure>`_, which can help to isolate the cause.


Delete or rewrite the test
~~~~~~~~~~~~~~~~~~~~~~~~~~

If the functionality is covered by other tests, perhaps the test can be removed. If not, perhaps it can be rewritten at a lower level which will remove the flakiness or make its source more apparent.


Quarantine
~~~~~~~~~~

Mark Lapierre discusses the `Pros and Cons of Quarantined Tests <https://dev.to/mlapierre/pros-and-cons-of-quarantined-tests-2emj>`_ in a post from 2018.



CI tools that rerun on failure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Azure Pipelines (the Azure cloud CI/CD tool, formerly Visual Studio Team Services or VSTS) has a feature to `identify flaky tests <https://docs.microsoft.com/en-us/previous-versions/azure/devops/2017/dec-11-vsts?view=tfs-2017#identify-flaky-tests>`_ and rerun failed tests.



Research
^^^^^^^^

This is a limited list, please submit an issue or pull request to expand it!

* Gao, Zebao, Yalan Liang, Myra B. Cohen, Atif M. Memon, and Zhen Wang. "Making system user interactive tests repeatable: When and what should we control?." In *Software Engineering (ICSE), 2015 IEEE/ACM 37th IEEE International Conference on*, vol. 1, pp. 55-65. IEEE, 2015.  `PDF <http://www.cs.umd.edu/~atif/pubs/gao-icse15.pdf>`__
* Palomba, Fabio, and Andy Zaidman. "Does refactoring of test smells induce fixing flaky tests?." In *Software Maintenance and Evolution (ICSME), 2017 IEEE International Conference on*, pp. 1-12. IEEE, 2017. `PDF in Google Drive <https://drive.google.com/file/d/10HdcCQiuQVgW3yYUJD-TSTq1NbYEprl0/view>`__
* Bell, Jonathan, Owolabi Legunsen, Michael Hilton, Lamyaa Eloussi, Tifany Yung, and Darko Marinov. "DeFlaker: Automatically detecting flaky tests." In *Proceedings of the 2018 International Conference on Software Engineering*. 2018. `PDF <https://www.jonbell.net/icse18-deflaker.pdf#section-Research>`__
* Dutta, Saikat and Shi, August and Choudhary, Rutvik and Zhang, Zhekun and Jain, Aryaman and Misailovic, Sasa. "Detecting flaky tests in probabilistic and machine learning applications." In *Proceedings of the 29th ACM SIGSOFT International Symposium on Software Testing and Analysis (ISSTA)*, pp. 211-224. ACM, 2020. `PDF <https://www.cs.cornell.edu/~saikatd/papers/flash-issta20.pdf>`__
* Habchi, Sarra and Haben, Guillaume and Sohn, Jeongju and Franci, Adriano and Papadakis, Mike and Cordy, Maxime and Le Traon, Yves. "What Made This Test Flake? Pinpointing Classes Responsible for Test Flakiness." In Proceedings of the 38th IEEE International Conference on Software Maintenance and Evolution (ICSME), IEEE, 2022. `PDF <https://arxiv.org/abs/2207.10143>`__
* Lamprou, Sokrates. "Non-deterministic tests and where to find them: Empirically investigating the relationship between flaky tests and test smells by examining test order dependency." Bachelor thesis, Department of Computer and Information Science, Linköping University, 2022. LIU-IDA/LITH-EX-G–19/056–SE. `PDF <https://www.diva-portal.org/smash/get/diva2:1713691/FULLTEXT01.pdf>`__
* Leinen, Fabian and Elsner, Daniel and Pretschner, Alexander and Stahlbauer, Andreas and Sailer, Michael and Jürgens, Elmar. "Cost of Flaky Tests in Continuous Integration: An Industrial Case Study." Technical University of Munich and CQSE GmbH, Munich, Germany, 2023. `PDF <https://mediatum.ub.tum.de/doc/1730194/1730194.pdf>`__

Resources
^^^^^^^^^

* `Eradicating Non-Determinism in Tests <https://martinfowler.com/articles/nonDeterminism.html>`_ by Martin Fowler, 2011
* `No more flaky tests on the Go team <https://www.thoughtworks.com/insights/blog/no-more-flaky-tests-go-team>`_ by Pavan Sudarshan, 2012
* `The Build That Cried Broken: Building Trust in your Continuous Integration Tests <https://www.youtube.com/embed/VotJqV4n8ig>`_ talk (video) by `Angie Jones <https://angiejones.tech/>`_ at SeleniumConf Austin 2017
* `Test and Code Podcast: Flaky Tests and How to Deal with Them <https://testandcode.com/50>`_ by Brian Okken and Anthony Shaw, 2018
* Microsoft:

  * `How we approach testing VSTS to enable continuous delivery <https://blogs.msdn.microsoft.com/bharry/2017/06/28/testing-in-a-cloud-delivery-cadence/>`_ by Brian Harry MS, 2017
  * `Eliminating Flaky Tests <https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/eliminating-flaky-tests>`_ blog and talk (video) by Munil Shah, 2017

* Google:

  * `Flaky Tests at Google and How We Mitigate Them <https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html>`_ by John Micco, 2016
  * `Where do Google's flaky tests come from? <https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html>`_  by Jeff Listfield, 2017

* Dropbox:
  * `Athena: Our automated build health management system <https://dropbox.tech/infrastructure/athena-our-automated-build-health-management-system>`_ by Utsav Shah, 2019
  * `How To Manage Flaky Tests in your CI Workflows <https://mill-build.org/blog/4-flaky-tests.html>`_ by Li Haoyi, 2025

* Uber:
  * `Handling Flaky Unit Tests in Java <https://www.uber.com/blog/handling-flaky-tests-java/>`_ by Uber Engineering, 2021
  * `Flaky Tests Overhaul at Uber <https://www.uber.com/blog/flaky-tests-overhaul/>`_ by Uber Engineering, 2024

.. _pytest-xdist: https://github.com/pytest-dev/pytest-xdist
