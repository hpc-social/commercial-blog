---
author: RedHat Hybrid cloud blog
author_tag: redhat
blog_subtitle: Red Hat open hybrid cloud blog
blog_title: Hybrid cloud blog
blog_url: https://content.cloud.redhat.com/blog
category: redhat
date: '2022-12-20 14:00:00'
layout: post
original_url: https://content.cloud.redhat.com/blog/how-to-version-bound-images-and-artifacts-for-openshift-operators
slug: how-to-version-bound-images-and-artifacts-for-openshift-operators
title: How to Version-Bound Images and Artifacts for OpenShift Operators
---

<div class="hs-featured-image-wrapper"> 
 <a class="hs-featured-image-link" href="https://content.cloud.redhat.com/blog/how-to-version-bound-images-and-artifacts-for-openshift-operators" title=""> <img alt="How to Version-Bound Images and Artifacts for OpenShift Operators" class="hs-featured-image" src="https://content.cloud.redhat.com/hubfs/NMAH-JN2014-3618.jpg" style="width: auto !important; float: left; margin: 0 15px 15px 0;" /> </a> 
</div>
  
<div> 
 <h2>Background/Purpose</h2> 
 <p>A common practice in Telco environments is to rely on using an<span>&nbsp;</span><strong>Offline Registry</strong><span>&nbsp;</span>to store all the container base images for OpenShift deployments. This Offline Registry needs to meet some requirements, and in this article we will focus mainly on how to optimize the storage used by the Offline Registry. </p>
 
 <p>❗Be advised that the following values are an example representation for the purpose of this article, in your case the values might differ.</p>
 
 <h2>Environment Setup</h2> 
 <p>The environment we are using consists of one bare-metal host whose role is the Bastion Node or Provisioning Node, DHCP-server and DNS-server, which will also host the Offline Registry and RHCOS-cache-httpd-server.</p>
 
 <p>The details contained in this article are independent of the installer used to deploy the cluster.</p>
 
 <h2>Step 0. Installing the required tools</h2> 
 <p>Download the<span>&nbsp;</span><strong>oc-mirror</strong><span>&nbsp;</span>cli:</p>
 
 <div> 
  <div> 
   <pre>$ export VERSION=stable-4.11<br />$ curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/oc-mirror.tar.gz | tar zxvf - oc-mirror<br />$ sudo cp oc-mirror /usr/local/bin<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p>As described in<span>&nbsp;</span><a href="https://docs.openshift.com/container-platform/4.10/installing/disconnected_install/installing-mirroring-disconnected.html">here</a>, the oc-mirror cli becomes GA in channel-4.11.</p>
 
 <p>The use of the oc-mirror cli is independent of the Offline Registry used (eg. Quay, docker registry, JFROG Artifactory, etc).</p>
 
 <h2>Step 1. Building the imageset-config.yaml</h2> 
 <h3>Step 1.1 How to check the operator version included in the redhat-operator-index channel</h3> 
 <p>In this section we are going to introduce the procedure for obtaining the specific operator version we are going to use in the next section.</p>
 
 <p>We are going to run the redhat-operator-index for tag 4.10 as a rootless podman:</p>
 
 <div> 
  <div> 
   <pre>$ mkdir -p ${HOME}/.config/systemd/user<br />$ podman login registry.redhat.io<br />$ podman run -d --name redhat-operator-index-4.10 -p 50051:50051 -it registry.redhat.io/redhat/redhat-operator-index:v4.10<br />$ cd ${HOME}/.config/systemd/user/<br />$ podman generate systemd --name  redhat-operator-index-4.10 &gt;&gt; container-redhat-operator-index.service<br />$ systemctl --user daemon-reload<br />$ systemctl --user enable container-redhat-operator-index.service<br />$ systemctl --user restart container-redhat-operator-index.service<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p>Validate if the container its running:</p>
 
 <div> 
  <div> 
   <pre>$ podman ps<br />CONTAINER ID  IMAGE                                                  COMMAND               CREATED       STATUS          PORTS                     NAMES<br />ffe3352d17f9  registry.redhat.io/redhat/redhat-operator-index:v4.10  registry serve --...  7 weeks ago   Up 2 hours ago  0.0.0.0:50051-&gt;50051/tcp redhat-operator-index-4.10</pre> 
  </div>
 
 </div>
 
 <p>By creating this container, we will be able to check the content of the channel content for OCPv4.10.</p>
 
 <p>To determine the versions available we need to query the redhat-operator-index endpoint:</p>
 
 <div> 
  <div> 
   <pre>$ export LOCAL_RH_OPERATOR_INDEX=inbacrnrdl0101.offline.redhat.lan<br />$ export LOCAL_RH_OPERATOR_INDEX_PORT=50051<br />$ grpcurl -plaintext  ${LOCAL_RH_OPERATOR_INDEX}:${LOCAL_RH_OPERATOR_INDEX_PORT} api.Registry.ListBundles | jq ' .packageName, .channelName, .bundlePath, .version'<br />…<br />parts of the output omitted<br />…<br />"odf-operator"<br />"stable-4.10"<br />"registry.redhat.io/odf4/odf-operator-bundle@sha256:662ec108960703f41652ff47b49e6a509a52fe244d52891d320b821dd9521f55"<br />"4.10.4"<br />"odf-operator"<br />"stable-4.10"<br />"registry.redhat.io/odf4/odf-operator-bundle@sha256:182d966cb488b188075d2ffd3f6451eec179429ac4bff55e2e26245049953a82"<br />"4.10.5"<br />…<br />parts of the output omitted<br />…<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p>The grpcurl binary has been obtained from<span>&nbsp;</span><a href="https://github.com/fullstorydev/grpcurl/releases">here</a></p>
 
 <h3>Step 1.2. How to build a Offline Registry [Optional]</h3> 
 <p>In this section we are going to highlight an example on how to create an Offline Registry that can be used to highlight the principle of mirroring the container base images.</p>
 
 <p>Creating the working directory of the Offline Registry:</p>
 
 <div> 
  <div> 
   <pre>$ mkdir -p ${HOME}/registry/{auth,certs,data}</pre> 
  </div>
 
 </div>
 
 <p>Creating the username and password used by the Offline Registry:</p>
 
 <div> 
  <div> 
   <pre>$ htpasswd -bBc ${HOME}/registry/auth/htpasswd &lt;username&gt;&lt;password&gt;<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p>Please, note that the values for the<span>&nbsp;</span>and<span>&nbsp;</span>should be updated with your particular ones.</p>
 
 <p>Creating the certificate used by the Offline Registry:</p>
 
 <div> 
  <div> 
   <pre>$ export host_fqdn=inbacrnrdl0101.offline.redhat.lan<br />$ cert_c="AT" <br />$ cert_s="WIEN"<br />$ cert_l="WIEN"<br />$ cert_o="TelcoEngineering"<br />$ cert_ou="RedHat"<br />$ cert_cn="${host_fqdn}" <br />$ openssl req \<br />    -newkey rsa:4096 \<br />    -nodes \<br />    -sha256 \<br />    -keyout ${HOME}/registry/certs/domain.key \<br />    -x509 \<br />    -days 365 \<br />    -out ${HOME}/registry/certs/domain.crt \<br />    -addext "subjectAltName = DNS:${host_fqdn}" \<br />    -subj "/C=${cert_c}/ST=${cert_s}/L=${cert_l}/O=${cert_o}/OU=${cert_ou}/CN=${cert_cn}"<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p>Please, note that the values used in the certificate creation should be updated with your particular ones. Start the Offline Registry container:</p>
 
 <div> 
  <div> 
   <pre>$ podman run -d --name ocpdiscon-registry -p 5050:5000 \<br />-e REGISTRY_AUTH=htpasswd \<br />-e REGISTRY_AUTH_HTPASSWD_REALM=Registry \<br />-e REGISTRY_HTTP_SECRET=ALongRandomSecretForRegistry \<br />-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \<br />-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \<br />-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \<br />-e REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true \<br />-e REGISTRY_STORAGE_DELETE_ENABLED=true \<br />-v ${HOME}/registry/data:/var/lib/registry:z \<br />-v ${HOME}/registry/auth:/auth:z \<br />-v ${HOME}/registry/certs:/certs:z docker.io/library/registry:2.8.1<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p>Based on the version list determined section<span>&nbsp;</span><strong>Step 1.1 How to check the operator version included in the redhat-operator-index channel</strong><span>&nbsp;</span>we are going to build the<span>&nbsp;</span><strong>imageset-config.yaml</strong><span>&nbsp;</span>in order to mirror the container base images.</p>
 
 <h3>Step 1.3. Building the credential file for the mirroring process</h3> 
 <p>In this section we are going to build the config.json file used in the mirroring process by the oc-mirror cli to gain authorization to the registry.redhat.io and to the Offline Registry.</p>
 
 <p>In the Offline Registry director create the config.json file:</p>
 
 <div> 
  <div> 
   <pre>$ touch ${HOME}/registry/config.json<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p><span>Open the browser and go to the following&nbsp;</span><a href="https://console.redhat.com/openshift/install/pull-secret">link</a><span>. As described in&nbsp;</span><a href="https://docs.openshift.com/container-platform/4.10/openshift_images/managing_images/using-image-pull-secrets.html">here</a><span>, to obtain the pull-secret.json file which we are going to edit and save it under the config.json. </span></p>
 
 <p>The config.json file structure should be close to the following format:</p>
 
 <div> 
  <div> 
   <pre>{<br />  "auths": {<br />    "cloud.openshift.com": {<br />      "auth": "&lt;base64-secret&gt;"<br />    },<br />    "inbacrnrdl0101.offline.redhat.lan:5051": {<br />      "auth": "&lt;base64-secret&gt;"<br />    },<br />    "quay.io": {<br />      "auth": "&lt;base64-secret&gt;"<br />    },<br />    "registry.connect.redhat.com": {<br />      "auth": "&lt;base64-secret&gt;"<br />    },<br />    "registry.fedoraproject.org": {<br />      "auth": "&lt;base64-secret&gt;"<br />    },<br />    "registry.redhat.io": {<br />      "auth": "&lt;base64-secret&gt;"<br />    }<br />  }<br />}</pre> 
  </div>
 
 </div>
 
 <p>Now we will have to let oc-mirror cli use the config.json file:</p>
 
 <div> 
  <div> 
   <pre><code><span>$ </span><span>export </span><span>DOCKER_CONFIG</span><span>=</span><span>${</span><span>HOME</span><span>}</span>/registry/config.json<br /></code></pre> 
  </div>
 
 </div>
 
 <h2>Step 2. How to control the storage usage of the mirror</h2> 
 <h3>Step 2.1. How to mirror container base images with a compact filesystem usage</h3> 
 <p>Mirror the container base images operator to the .tar file under the archive directory:</p>
 
 <div> 
  <div> 
   <pre>$ oc-mirror --config imageset-config.yaml file://archive</pre> 
  </div>
 
 </div>
 
 <p>The content of a sample imageset-config.yaml file to be used in command above are :</p>
 
 <div> 
  <div> 
   <pre>$ cat imageset-config.yaml<br />apiVersion: mirror.openshift.io/v1alpha2<br />kind: ImageSetConfiguration<br />mirror:<br />  operators:<br />    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.10<br />      targetName: 'rh-index'<br />      targetTag: v1-test<br />      full: false<br />      packages:<br />        - name: odf-operator<br />      packages:<br />        - name: odf-operator<br />          minVersion: '4.10.4'<br />          maxVersion: '4.10.4'<br />          channels:<br />                  - name: 'stable-4.10'</pre> 
  </div>
 
 </div>
 
 <p>Check the archive mirror_seq1_000000.tar size after the entire mirroring process has been finished:</p>
 
 <div> 
  <div> 
   <pre>$ du -h ./archive/mirror_seq1_000000.tar<br />6.2G    ./archive/mirror_seq1_000000.tar</pre> 
  </div>
 
 </div>
 
 <p>Before proceeding with any kind of mirroring steps, we are going to have a closer look at the Offline Registry status:</p>
 
 <div> 
  <div> 
   <pre>$ tree ${HOME}/registry/<br />registry/<br />├── auth<br />│   └── htpasswd<br />├── certs<br />│   ├── domain.crt<br />│   └── domain.key<br />└── data<br /><br />3 directories, 3 files</pre> 
  </div>
 
 </div>
 
 <p>As we can observe, the ./data/ directory is empty at this point and we are going to proceed with the container base image mirroring procedure.</p>
 
 <p>Mirror the container base images from the file mirror_seq1_000000.tar to the offline registry.</p>
 
 <div> 
  <div> 
   <pre>$ export REGISTRY_NAME=inbacrnrdl0101.offline.redhat.lan<br />$ export REGISTRY_NAMESPACE=olm-mirror<br />$ export REGISTRY_PORT=5050<br />$ oc-mirror --from ./archive docker://${REGISTRY_NAME}:${REGISTRY_PORT}/${REGISTRY_NAMESPACE}</pre> 
  </div>
 
 </div>
 
 <p>Check the content of the offline registry:</p>
 
 <div> 
  <div> 
   <pre>$ curl -X GET -u &lt;username&gt;:&lt;password&gt; https://${REGISTRY_NAME}:${REGISTRY_PORT}/v2/_catalog --insecure | jq .<br />{<br />  "repositories": [<br />    "karmab/curl",<br />    "karmab/kubectl",<br />    "karmab/mdns-publisher",<br />    "karmab/origin-coredns",<br />    "karmab/origin-keepalived-ipfailover",<br />    "ocp-release",<br />    "olm-mirror/odf4/cephcsi-rhel8",<br />    "olm-mirror/odf4/mcg-core-rhel8",<br />    "olm-mirror/odf4/mcg-operator-bundle",<br />    "olm-mirror/odf4/mcg-rhel8-operator",<br />    "olm-mirror/odf4/ocs-must-gather-rhel8",<br />    "olm-mirror/odf4/ocs-operator-bundle",<br />    "olm-mirror/odf4/ocs-rhel8-operator",<br />    "olm-mirror/odf4/odf-console-rhel8",<br />    "olm-mirror/odf4/odf-csi-addons-operator-bundle",<br />    "olm-mirror/odf4/odf-csi-addons-rhel8-operator",<br />    "olm-mirror/odf4/odf-csi-addons-sidecar-rhel8",<br />    "olm-mirror/odf4/odf-operator-bundle",<br />    "olm-mirror/odf4/odf-rhel8-operator",<br />    "olm-mirror/odf4/rook-ceph-rhel8-operator",<br />    "olm-mirror/odf4/volume-replication-rhel8-operator",<br />    "olm-mirror/openshift4/ose-csi-external-attacher",<br />    "olm-mirror/openshift4/ose-csi-external-provisioner",<br />    "olm-mirror/openshift4/ose-csi-external-resizer",<br />    "olm-mirror/openshift4/ose-csi-external-snapshotter",<br />    "olm-mirror/openshift4/ose-csi-node-driver-registrar",<br />    "olm-mirror/openshift4/ose-kube-rbac-proxy",<br />    "olm-mirror/redhat/rh-index",<br />    "olm-mirror/rhceph/rhceph-5-rhel8",<br />    "olm-mirror/rhel8/postgresql-12"<br />  ]<br />}</pre> 
  </div>
 
 </div>
 
 <p>As it can be observed in the above output, we have a header containing base images whose purpose is to be used in the OCP deployment, those images are not usable in the odf-operator installation. This header is the following:</p>
 
 <div> 
  <div> 
   <pre>$ curl -X GET -u &lt;username&gt;:&lt;password&gt; https://${REGISTRY_NAME}:${REGISTRY_PORT}/v2/_catalog --insecure | jq .                                     <br />{<br />  "repositories": [<br />    "karmab/curl",<br />    "karmab/kubectl",<br />    "karmab/mdns-publisher",<br />    "karmab/origin-coredns",<br />    "karmab/origin-keepalived-ipfailover",<br />    "ocp-release"<br />  ]<br />}</pre> 
  </div>
 
 </div>
 
 <p>This header’s filesystem usage:</p>
 
 <div> 
  <div> 
   <pre>$ du -h ${HOME}/registry/data/ --max-depth=1<br />13G     registry/data/docker<br />13G     registry/data/<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p>Highlighted the content of the<span>&nbsp;</span><strong>odf-operator</strong><span>&nbsp;</span>mirrored. Now we are going to evaluate the file system used at this moment by the above content of the offline registry.</p>
 
 <div> 
  <div> 
   <pre>$ du -h ${HOME}/registry/data/ --max-depth=1<br />19G     registry/data/docker<br />19G     registry/data/<code><br /></code></pre> 
  </div>
 
 </div>
 
 <h3>Step 2.2. How to make use of the Offline Registry content to your OCP cluster</h3> 
 <p>Once the mirroring of the operators is finished the process is creating the following directory: oc-mirror-workspace/results-1667747309 for which we will use the following two files to apply it to the OCP cluster:</p>
 
 <div> 
  <div> 
   <pre>$ cat oc-mirror-workspace/results-1667747309/catalogSource-rh-index.yaml<br />apiVersion: operators.coreos.com/v1alpha1<br />kind: CatalogSource<br />metadata:<br />  name: rh-index<br />  namespace: openshift-marketplace<br />spec:<br />  image: inbacrnrdl0101.offline.redhat.lan:5050/olm-mirror/redhat/rh-index:v1-test<br />  sourceType: grpc</pre> 
  </div>
 
 </div>
 
 <div> 
  <div> 
   <pre>$ cat oc-mirror-workspace/results-1667747309/imageContentSourcePolicy.yaml<br />---<br />apiVersion: operator.openshift.io/v1alpha1<br />kind: ImageContentSourcePolicy<br />metadata:<br />  labels:<br />    operators.openshift.org/catalog: "true"<br />  name: operator-0<br />spec:<br />  repositoryDigestMirrors:<br />  - mirrors:<br />    - inbacrnrdl0101.offline.redhat.lan:5050/olm-mirror/openshift4<br />    source: registry.redhat.io/openshift4<br />  - mirrors:<br />    - inbacrnrdl0101.offline.redhat.lan:5050/olm-mirror/odf4<br />    source: registry.redhat.io/odf4<br />  - mirrors:<br />    - inbacrnrdl0101.offline.redhat.lan:5050/olm-mirror/rhel8<br />    source: registry.redhat.io/rhel8<br />  - mirrors:<br />    - inbacrnrdl0101.offline.redhat.lan:5050/olm-mirror/rhceph<br />    source: registry.redhat.io/rhceph<br />  - mirrors:<br />    - inbacrnrdl0101.offline.redhat.lan:5050/olm-mirror/redhat<br />    source: registry.redhat.io/redhat<code><br /></code></pre> 
  </div>
 
 </div>
 
 <p><a href="https://content.cloud.redhat.com/hubfs/pic1.png"></a></p>
 
 <p>Looking at the applied catalog source, we can observe that the endpoint corresponds to the offline registry used: inbacrnrdl0101.offline.redhat.lan:5050.</p>
 
 <p><a href="https://content.cloud.redhat.com/hubfs/pic1.png"></a></p>
 
 <p>In order to validate that there are no dependencies missing for the odf-mirror we will proceed installing the operator on the OCP cluster.</p>
 
 <ul> 
  <li>Start the installation of<span>&nbsp;</span><strong>odf-operator</strong>:</li> 
 </ul> 
 <ul> 
  <li><strong>odf-operator</strong><span>&nbsp;</span>is installed:</li> 
 </ul> 
 <p><a href="https://content.cloud.redhat.com/hubfs/pic5.png"></a></p>
 
 <ul> 
  <li>The installed<span>&nbsp;</span><strong>odf-operator</strong><span>&nbsp;</span>subscription used:</li> 
 </ul> 
 <ul> 
  <li><strong>odf-operator</strong><span>&nbsp;</span>pods status:</li> 
 </ul> 
 <div> 
  <div> 
   <pre>$ oc get pods -n openshift-storage<br />NAME                                               READY   STATUS    RESTARTS      AGE<br />csi-addons-controller-manager-78c75c4c7-wq4p5      2/2     Running   2 (37m ago)   77m<br />noobaa-operator-6fdd894554-ptnm9                   1/1     Running   0             78m<br />ocs-metrics-exporter-775b6d4bdf-k5d52              1/1     Running   0             78m<br />ocs-operator-6bf7b6dfc6-zwwzv                      1/1     Running   2 (37m ago)   78m<br />odf-console-6dff658495-dcfxh                       1/1     Running   0             78m<br />odf-operator-controller-manager-54ddd5db9c-mpkwt   2/2     Running   2 (37m ago)   78m<br />rook-ceph-operator-57bfbcc9d-hb9tk                 1/1     Running   0             78m<code><br /></code></pre> 
  </div>
 
 </div>
 
 <h3>Step 2.3. How to mirror container base images with a uncompact filesystem usage</h3> 
 <p>Now we evaluate the filesystem usage when the imageset-config.yaml parameter full: true is used, the imageset-config.yaml file will have the following content:</p>
 
 <div> 
  <div> 
   <pre>$ cat imageset-config.yaml<br />apiVersion: mirror.openshift.io/v1alpha2<br />kind: ImageSetConfiguration<br />mirror:<br />  operators:<br />    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.10<br />      targetName: 'rh-index'<br />      targetTag: v1-test<br />      full: true<br />      packages:<br />        - name: odf-operator<br />      packages:<br />        - name: odf-operator<br />          minVersion: '4.10.4'<br />          maxVersion: '4.10.4'<br />          channels:<br />                  - name: 'stable-4.10'</pre> 
  </div>
 
 </div>
 
 <p>Proceed to mirror the&nbsp;odf-operator&nbsp;container base images to the .tar file as highlighted above:</p>
 
 <div> 
  <div> 
   <pre>$ oc-mirror --config imageset-config.yaml file://archive</pre> 
  </div>
 
 </div>
 
 <p>Once the mirroring has completed, validate the .tar file size in comparison with the previous check:</p>
 
 <div> 
  <div> 
   <pre>$ du -h ./archive/mirror_seq1_000000.tar<br />32G     ./archive/mirror_seq1_000000.tar<code><br /></code></pre> 
  </div>
 
 </div>
 
 <ul> 
  <li>Filesystem usage of the .tar file comparison:</li> 
 </ul> 
 <table style="border-collapse: collapse; margin-left: auto; margin-right: auto; border: 1px solid #99acc2;"> 
  <thead> 
   <tr> 
    <th>imageset-config.yaml differences</th> 
    <th>mirror_seq1_000000.tar [Gb]</th> 
    <th>Notes</th> 
   </tr> 
  </thead> 
  <tbody> 
   <tr> 
    <td>with full: true</td> 
    <td>32</td> 
    <td>&nbsp;</td> 
   </tr> 
   <tr> 
    <td>with full: false</td> 
    <td>6.2</td> 
    <td>80.625% decrease</td> 
   </tr> 
  </tbody> 
 </table> 
 <p>We can observe from the .tar file size that its size was reduced by 80% for the same content as in the previous,<span>&nbsp;</span><strong>Step 2.1. How to mirror container base images with a compact filesystem usage example</strong>.</p>
 
 <p>Mirror the container base images from the newly created file mirror_seq1_000000.tar to the offline registry. Be advised that the following values are an example representation for the purpose of this article, in your case the values might differ.</p>
 
 <div> 
  <div> 
   <pre>$ export REGISTRY_NAME=inbacrnrdl0101.offline.redhat.lan<br />$ export REGISTRY_NAMESPACE=olm-mirror<br />$ export REGISTRY_PORT=5050<br />$ oc-mirror --from ./archive docker://${REGISTRY_NAME}:${REGISTRY_PORT}/${REGISTRY_NAMESPACE}</pre> 
  </div>
 
 </div>
 
 <p>Check the content of the offline registry:</p>
 
 <div> 
  <div> 
   <pre>$ curl -X GET -u &lt;username&gt;:&lt;password&gt; https://${REGISTRY_NAME}:${REGISTRY_PORT}/v2/_catalog --insecure | jq .<br />{<br />  "repositories": [<br />    "karmab/curl",<br />    "karmab/kubectl",<br />    "karmab/mdns-publisher",<br />    "karmab/origin-coredns",<br />    "karmab/origin-keepalived-ipfailover",<br />    "ocp-release",<br />    "olm-mirror/odf4/cephcsi-rhel8",<br />    "olm-mirror/odf4/mcg-core-rhel8",<br />    "olm-mirror/odf4/mcg-operator-bundle",<br />    "olm-mirror/odf4/mcg-rhel8-operator",<br />    "olm-mirror/odf4/ocs-must-gather-rhel8",<br />    "olm-mirror/odf4/ocs-operator-bundle",<br />    "olm-mirror/odf4/ocs-rhel8-operator",<br />    "olm-mirror/odf4/odf-console-rhel8",<br />    "olm-mirror/odf4/odf-csi-addons-operator-bundle",<br />    "olm-mirror/odf4/odf-csi-addons-rhel8-operator",<br />    "olm-mirror/odf4/odf-csi-addons-sidecar-rhel8",<br />    "olm-mirror/odf4/odf-operator-bundle",<br />    "olm-mirror/odf4/odf-rhel8-operator",<br />    "olm-mirror/odf4/rook-ceph-rhel8-operator",<br />    "olm-mirror/odf4/volume-replication-rhel8-operator",<br />    "olm-mirror/openshift4/ose-csi-external-attacher",<br />    "olm-mirror/openshift4/ose-csi-external-provisioner",<br />    "olm-mirror/openshift4/ose-csi-external-resizer",<br />    "olm-mirror/openshift4/ose-csi-external-snapshotter",<br />    "olm-mirror/openshift4/ose-csi-node-driver-registrar",<br />    "olm-mirror/openshift4/ose-kube-rbac-proxy",<br />    "olm-mirror/redhat/rh-index",<br />    "olm-mirror/rhceph/rhceph-5-rhel8",<br />    "olm-mirror/rhel8/postgresql-12"<br />  ]<br />}</pre> 
  </div>
 
 </div>
 
 <p>We can observe that the content of the offline registry is identical.</p>
 
 <p>Validate the file system used by the container base images mirrored to the offline registry:</p>
 
 <div> 
  <div> 
   <pre>$ du -h ${HOME}/registry/data/ --max-depth=1<br />44G     registry/data/docker<br />44G     registry/data/<code><br /></code></pre> 
  </div>
 
 </div>
 
 <ul> 
  <li>Filesystem usage of the uncompressed container base images usage comparison:</li> 
 </ul> 
 <table style="border-collapse: collapse; margin-left: auto; margin-right: auto; border: 1px solid #99acc2;"> 
  <thead> 
   <tr> 
    <th>imageset-config.yaml differences</th> 
    <th>mirror_seq1_000000.tar [Gb]</th> 
    <th>Notes</th> 
   </tr> 
  </thead> 
  <tbody> 
   <tr> 
    <td>with full: true</td> 
    <td>44</td> 
    <td>&nbsp;</td> 
   </tr> 
   <tr> 
    <td>with full: false</td> 
    <td>19</td> 
    <td>56.8182% decrease</td> 
   </tr> 
  </tbody> 
 </table> 
 <p>We can observe from the offline registry container base image content size was optimized with 56.8182% for the same content in the previous example.</p>
 
 <h2>Conclusions</h2> 
 <p>In conclusion, by leveraging the version control and restricting the container base image download we are able to optimize the filesystem usage of the offline registry.</p>
 
</div>
  
<img alt="" height="1" src="https://track.hubspot.com/__ptq.gif?a=4305976&amp;k=14&amp;r=https%3A%2F%2Fcontent.cloud.redhat.com%2Fblog%2Fhow-to-version-bound-images-and-artifacts-for-openshift-operators&amp;bu=https%253A%252F%252Fcontent.cloud.redhat.com%252Fblog&amp;bvt=rss" style="width: 1px!important;" width="1" />