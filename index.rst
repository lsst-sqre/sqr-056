#############################################################################
Guidelines for gated updates for SQuaRE services (including Science Platform)
#############################################################################

.. abstract::

   The aim of this document is to expose the heuristics by which we consider what the appropriate process is for various types of service updates. We also describe our extant processes around Science Platform and Science Platform adjacent services, as these are likely to be of most interest to developers outside our team.

..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.




.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   The aim of this document is to expose the heuristics by which we consider what the appropriate process is for various types of service updates.
   We also describe our extant processes around Science Platform and Science Platform adjacent services, as these are likely to be of most interest to developers outside our team.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. note::

   This is a *descriptive* (*not* prescriptive) document describing details of current practice of a specific team in a specific area. For the actual developer guide, see developer.lsst.io


Background
==========

SQuaRE uses Configuration as Code (aka “gitops”) practices for service deployment whenever practical.
What this means is that by merging code or configuration to a special branch (typically master), it is *possible* to trigger an action (via an agent or supervisory system) that automatically results in the deployment of the new version of the service.

Whether the new version of the service is *actually* deployed automatically (a practice known as continuous deployment) or whose deployment is gated behind a human decision making process is a matter of policy.
The barrier to getting through that gate is itself a function of where a system lies on an axis of formality, with one end maximizing developer agency and the other maximizing organisational control.

.. figure:: _static/axis.png
   :name: fig-axis

   Some common examples along the axis of formality

Where any one particular instance of any one particular service lies in the Axis of Formality is a function of:

- The service’s maturity, including the number of users depending on the service

- The service environment, including whether it is production, staging or development environment

- The service’s userbase, including whether it is a feature-seeking or risk-averse population

- The actual and reputational impact of disruptions to the service

- The nature of the change being made.

It follows that a major change in a mature high visibility service that many users depend on is on the opposite side of the axis that a cosmetic change in a service under development deployed on a sandbox.
The trickier situations lie in between these extremes, and the aim of this document is to expose the heuristics by which we determine these questions.
We also describe our extant processes around the update of Science Platform and Science Platform adjacent services, as these are likely to be of most interest to developers outside our team and since they exercise a lot of the decision space described above.

Kubernetes services and Argo CD
===============================

SQuaRE develops Kubernetes-based services, and uses the ArgoCD platform to manage their deployment. The ArgoCD UI provides a valuable way both to assess the state of deployed services and to allow basic maintenance operations on deployed services even without being particularly intimate with the details of their deployments, and irrespective whether the underlying service is configured with Helm or kustomize.

.. figure:: _static/argocd.png
   :name: fig-argocd

   The overview panel of the ArgoCD UI

ArgoCD continuously monitors running services and compares their state with that of their git repo. When it detects that a service is “out of sync” (ie there is drift between its deployed state and the repository state) it can sync automatically (the continuous deployment model) or as in Figure 2, indicate visually the out-of-sync state until an engineer pushes the sync button to resolve it (the gated deployment model).
Generally unless working in a dev or bleed environment, we do not allow ArgoCD to sync automatically.

It follows then that for gated deployments there are two aspects to getting a new version of a service deployed into production:

1. Mechanics: Getting the right version of the service and its configuration merged into the deployment branch (typically master);
2. Process: Going through the process that allows it to be synced into production.

Mechanics
=========

Assume that there is a new release of Gafaelfawr, the authentication service used by the Rubin Science Platform.
The following steps notify Argo CD that this upgrade is ready to be performed:

#. Update the Helm chart in the [charts](https://github.com/lsst-sqre/charts) repository under `charts/gafaelfawr`.
   Do this by updating the `appVersion` key in `Chart.yaml` and the `image.tag` key in `values.yaml` to point to the new release.
   Also update the version of the Helm chart in `Chart.yaml`.
   When this PR is merged, a new version of the chart will be automatically published via GitHub Actions.
#. Update the Argo CD configuration to deploy the new version of the chart.
   The `Chart.yaml` file in `services/gafaelfawr` in the [phalanx](https://github.com/lsst-sqre/phalanx) repository determines what version of the Gafaelfawr chart is installed by Argo CD.
   (Phalanx is the Argo CD configuration repository for the Rubin Science Platform.)
   Under the stanza for the `gafaelfawr` chart in the `dependencies` key, change the version to match the newly-released chart version.
   Once this PR is merged, Argo CD will notice there is a pending upgrade for Gafaelfawr.

Process
=======

The process used to deploy a specific instance of a service depends on where it lies on the axis of formality above. We use the following terminology for the various deployment environments, in decreasing formality:

-  **Prod:** Production, a user-facing deployment. For historical reasons sometimes referred to as “stable” where the Science Platform is concerned.

-  **Int:** Integration, a deployment for developers, typically from different teams, to converge upon to test the integration of services prior to deployment on Prod. This is the environment somethings referred to as “staging”.

-  **Dev:** Development, an instance that is primarily for developer testing. It may not be always available, or there may be several of them.

-  **Bleed:** An environment that is left uncontrolled, either by being continuously deployed from master or by letting otherwise pinned versions of components float up

The line between Dev and Int can be different depending on the environment and its stakeholder teams.
For example data-dev.lsst.cloud is strictly reserved for (mainly) SQuaRE developers working on service infrastructure and other development that can result in core services being non-functional; meanwhile science platform application developers target data-int.lsst.cloud as they can work on their application on an otherwise stable environment.
On the other hand, the Dev system for telescope services (tucson-teststand.lsst.codes) is treated more carefully by SQuaRE developers to avoid interfering with telescope service infrastucture work.

In some cases work is corralled in scheduled maintenance windows.
Reasons for this include:

-  To minimize the potential of disruption in high availability environments

-  To allow co-ordination of work on multiple services with inter-dependencies and/or infrastructure

-  To communicate a “hold off reporting problems” message to users in order to avoid the report of transient issues associated with the upgrade

-  To enable the "Patch day call" format where maintainers of environments not operated by SQuaRE can get on zoom with us to roll out  updates in "pair programming" mode for easy access to help if they have any questions or unanticipated problems due to peculiarities of their IT infrastructure

*Maintenance windows do not imply fixed downtime.*
Downtime (complete service unavailability) is extremely rare and we design our processes to avoid it.
Routine maintenance work involves transient service unavailability at most and in most cases users are not barred from using the system during that time, though they are given notice to save their work and there is always a small chance of unforseen problems.
In some environments a co-ordinator is assigned to announce work start and work end and field any questions.

Current fixed maintenance windows (for applicable services/deployments) are:

-  **Summit Window:** Typically 1st Wednesday of every month during lunchtime at the observatory summit (13:00 Chile local), confirmed with the telescope software configuration manager. Environments: Tucson Teststand (dev), (La Serena) Base (int), Summit (prod).

-  **Patch Thursday:** Weekly, Thursday afternoons (15:00 Project/Pacific). Environments: Science RSP clusters (data.lsst.cloud and its -dev and -int analogues), Staff RSP clusters (usdf-rsp.slac.stanford.edu etc), developer services cluster (roundtable.lsst.cloud and its dev).

Again, these are not scheduled downtimes. In the event that extended service downtime is needed in a production service (extremely rare), work is scheduled  with ample notice and co-ordination with stakeholders and/or at a time where disruption is minimized.

Here is a chart showing the current settled-upon practice in select areas:


.. raw:: html
   :file: table1.html

Container Environments in the Science Platform
===============================================

The above discussion pertains to services, ie codebases where an error could affect a service's availability. When it comes to containers made available _by_ a service (eg in nublado), we are less risk averse as users, by design, can always fall back on a previously usable container in case of problems. We recognize that the Science Platform is a primary user environment and as such users do not wish to wait a week for some process to take its course in order to get a requested package or feature. We currently (and anticipate continuing to do so) provide containers labeled "experimental" to rapidly service ad-hoc user requests, for example, in addition to generally available daily builds.

How to reconcile this user-first orientation to the issue of scientific reproducibility is a matter for a future technote.

The Recommended Container
-------------------------

A special case of the nublado container is the image promoted by the spawner page as "recommended."
Since, by recommending a particular container to use, we take on a certain amount of responsibility to make sure that image is compatible with other services and materials (i.e. notebooks) we make available on the various RSP deployments.
The extra responisbility is met using a more rigorous process for promoting a new recommended tag.
The steps are:

#. Anyone can propose an image to be promoted to recommended when they feel there is sufficiently high value in new content since the previous recommended version.
   Typically this will be the Science Platform team, but architecture, for example, may have middleware reasons to propose an image.
#. Proposals for images to be promoted to recommended will be brought to the weekly RSP operations meeting.
#. If the proposal is viewed positively at the RSPOps meeting it is then brought to the weekly Data Preview Coordination meeting (or its eventual successor).
#. The Science Platform team iterates with the maintainers of first party notebook repositories (e.g. tutorial notebooks or notebooks intended to test functionality of the system) to make sure the notebooks run properly on all deployments where they will be provided.
#. At this point, a Jira ticket is opened by a member of the Science Platform team with appropriate watchers designated.
#. The ticket must be acknowledge by the following stake holders or their designates:

   - The lead for the Science Platfrom
   - The product owner for the Science Platform (or the Data Engineer in operations)
   - The lead for the Community Engagement Team
   - The lead for Science Pipelines
   - The lead for Middleware

#. During an advertised maintenance window, e.g. "Patch Thursday", the proposed image will be promoted as recommended.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa