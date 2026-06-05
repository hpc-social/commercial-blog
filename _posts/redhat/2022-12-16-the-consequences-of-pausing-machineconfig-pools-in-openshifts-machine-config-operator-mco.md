---
author: RedHat Hybrid cloud blog
author_tag: redhat
blog_subtitle: Red Hat open hybrid cloud blog
blog_title: Hybrid cloud blog
blog_url: https://content.cloud.redhat.com/blog
category: redhat
date: '2022-12-16 14:00:00'
layout: post
original_url: https://content.cloud.redhat.com/blog/the-consequences-of-pausing-machineconfig-pools-in-openshifts-machine-config-operator-mco
slug: the-consequences-of-pausing-machineconfig-pools-in-openshift-s-machine-config-operator-mco-
title: The Consequences Of Pausing MachineConfig Pools In OpenShift's Machine Config
  Operator (MCO)
---

<div class="hs-featured-image-wrapper"> 
 <a class="hs-featured-image-link" href="https://content.cloud.redhat.com/blog/the-consequences-of-pausing-machineconfig-pools-in-openshifts-machine-config-operator-mco" title=""> <img alt="The Consequences Of Pausing MachineConfig Pools In OpenShift's Machine Config Operator (MCO)" class="hs-featured-image" src="https://content.cloud.redhat.com/hubfs/POSTALHAT.png" style="width: auto !important; float: left; margin: 0 15px 15px 0;" /> </a> 
</div>
 
<p>The Machine Config Operator (MCO) in OpenShift exposes a "Pause" feature on its MachineConfigPools that allows users to halt config deployment. We made some changes in 4.11 to try to make that feature "safer", and this blog tries to give some context around what pausing a MachineConfigPool actually does under the hood, the consequences/tradeoffs, and a small glimpse into how the MCO team thinks about exposing and evolving features like this.</p>
  
<img alt="" height="1" src="https://track.hubspot.com/__ptq.gif?a=4305976&amp;k=14&amp;r=https%3A%2F%2Fcontent.cloud.redhat.com%2Fblog%2Fthe-consequences-of-pausing-machineconfig-pools-in-openshifts-machine-config-operator-mco&amp;bu=https%253A%252F%252Fcontent.cloud.redhat.com%252Fblog&amp;bvt=rss" style="width: 1px!important;" width="1" />