# Implemeting-a-Multi-Evironment-Application
here i will be submitting my project on implementing a multi- environment application 


Project Title: Implementing a Multi-Environment Application Deployment with Kustomize

1. Hypothetical Use Case:
   In this scenario, the mission is to roll out a lightweight web application to a Kubernetes cluster while supporting three distinct environments: development, staging, and production. Rather than maintain three separate sets of YAML files, the intention is to centralize all shared definitions in a base folder and then apply environment-specific changes through Kustomize overlays. Each overlay slightly alters behavior, such as the number of replicas, resource sizing, or environment variables, without repeating the entire configuration. The final outcome should let me run a single Kustomize command per environment and receive a complete, environment-tailored manifest. These manifests then become part of a CI/CD workflow so that any push or pull request automatically builds, applies, and verifies the configuration on a temporary test cluster.

2. Tasks:

Task 1: Set Up Your Project:
The objective here is to prepare a clean folder structure that Kustomize expects, separating common resources from per-environment modifications and ensuring everything is easy to navigate.

Step 1.1 (Task 1): I created a top-level directory called kustomize-capstone and moved into it. This folder serves as the project root for all subsequent files and subdirectories.

Step 1.2 (Task 1): Within the project root, I created a directory named base. This base folder will store the Kubernetes objects (like Deployment and Service) that are shared across all environments. The idea is that base remains stable, and overlays only tweak what differs.

Step 1.3 (Task 1): I created an overlays directory and then added three subfolders inside it: overlays/dev, overlays/staging, and overlays/prod. Each of these three folders will contain a kustomization.yaml that references ../../base and includes a small set of adjustments specific to that environment.

Step 1.4 (Task 1): I verified that my structure matches Kustomize conventions. At this point, the tree shows base, overlays/dev, overlays/staging, and overlays/prod, which confirms I’m set up to layer environment changes cleanly over the shared configuration.

Task 2: Initialize Git:
This task is about version control and preparing the repo so that any changes I make can be tracked, reviewed, and used to trigger the CI/CD pipeline.

Step 2.1 (Task 2): Inside the kustomize-capstone folder, I initialized a Git repository and ensured the primary branch is named main. This gives me a clean default branch for automation later.

Step 2.2 (Task 2): I configured my Git identity by setting my user name and email, so that commits clearly show who made each change and the CI has a consistent author to attribute changes to.

Step 2.3 (Task 2): I created a .gitignore file to ensure unnecessary files never enter the repository. Examples include macOS metadata files, editor settings folders, temporary build outputs, and any other local artifacts. I staged and committed the initial structure with these ignore rules so the history starts clean.

Task 3: Define Base Configuration:
The base configuration captures the shared, reusable Kubernetes resources and a Kustomize file that binds them together. It also demonstrates how to generate ConfigMaps and Secrets using Kustomize rather than hand-writing them.

Step 3.1 (Task 3): In base/deployment.yaml, I defined a Deployment named web. It includes a label app: web so the Service can target it. The Deployment specifies one replica by default, which overlays will later change for each environment. The Pod template runs a single container using the nginx:1.25-alpine image with port 80 exposed. I added one explicit environment variable, APP_ENV, initially set to the string base, because I plan for overlays to override this value to dev, staging, or prod. I configured a readiness probe that performs an HTTP GET against the root path on port 80 with a short initial delay and regular checks. Finally, I set conservative resource requests and limits to avoid overconsumption and to make scheduling predictable.

Step 3.2 (Task 3): In base/service.yaml, I created a ClusterIP Service named web that selects pods labeled app: web. It exposes port 80 as http and routes traffic to the container’s port 80. This Service allows other workloads in the cluster to reach the web pods by DNS name without exposing the service externally.

Step 3.3 (Task 3): In base/kustomization.yaml, I declared this directory as a Kustomization and listed deployment.yaml and service.yaml under resources so Kustomize knows to include them. I added commonLabels (for example, app.k8s.io/name and app.k8s.io/part-of) and a commonAnnotations entry (owner: dr-muogbo), so that every object generated from this base receives consistent metadata for discovery and ownership tracking. I used configMapGenerator to define a ConfigMap named app-config with two literals: WELCOME_MESSAGE set to “Hello from BASE” and FEATURE_FLAG_X set to “false.” I used secretGenerator to define a Secret named app-secret with an example literal API_TOKEN set to a harmless placeholder. I kept disableNameSuffixHash as false so that generated names include a content hash; this feature is important because changes to generator content produce new names, which triggers rolling updates safely. I then staged and committed the entire base directory so the repository permanently records these shared definitions.

Task 4: Create Environment-Specific Overlays:
For each environment (dev, staging, prod), the overlay imports base, gives every resource a distinct prefix, and uses a patch to adjust replicas, environment variables, and resource sizing. Each overlay also replaces the ConfigMap and Secret values with environment-tuned literals while keeping the generator names consistent.

Step 4.1 (Task 4: dev overlay): In overlays/dev, I wrote a patch file that modifies the web Deployment by setting replicas to 1, changing APP_ENV to “dev,” and defining resource requests and limits suitable for a development scenario. Then, in overlays/dev/kustomization.yaml, I pointed resources to ../../base, set namePrefix to dev- so names become dev-web, dev-app-config, and so on, and referenced the patch via patchesStrategicMerge. I also supplied configMapGenerator and secretGenerator sections with behavior: replace for the same generator names used in base. This tells Kustomize to take the base generator and entirely replace its values with dev-specific literals, such as WELCOME_MESSAGE for dev and a dev token placeholder.

Step 4.2 (Task 4: staging overlay): In overlays/staging, I created a similar deployment patch that sets replicas to 2, changes APP_ENV to “staging,” and sets moderate resource values appropriate for test environments that more closely mirror production. In overlays/staging/kustomization.yaml, I configured resources to reference ../../base, set namePrefix to stg- to prevent name collisions across environments, and included the staging deployment patch with patchesStrategicMerge. For configuration data, I again used behavior: replace under configMapGenerator and secretGenerator to supply staging-specific values for the welcome message and the API token placeholder.

Step 4.3 (Task 4: prod overlay): In overlays/prod, I wrote a patch that increases replicas to 3 and sets APP_ENV to “prod,” while also providing larger resource requests and limits that make sense for a production-like workload. In overlays/prod/kustomization.yaml, I listed ../../base as a resource, added namePrefix prod-, connected the prod patch using patchesStrategicMerge, and replaced the base generators with production literals. This ensures that production manifests are distinguishable, capacity-aware, and carry the correct configuration values for that environment. I committed the overlays directory so the environment-specific logic is fully tracked.

Task 5: Integrate with a CI/CD Pipeline:
The pipeline requirement is to set up an automated process that, on every push or pull request, checks out the repository, prepares the necessary tools, builds environment manifests with Kustomize, applies them to a temporary cluster, and confirms the Deployment becomes ready.

Step 5.1 (Task 5): I created a GitHub Actions workflow under .github/workflows and named the file deploy.yaml. The workflow runs on pushes to the main branch and on pull requests. It defines a job named deploy on an ubuntu-latest runner with a strategy matrix for env values dev, staging, and prod, so three jobs run in parallel each time the workflow triggers.

Step 5.2 (Task 5): Within the job steps, the first step checks out the repository using actions/checkout. Next, I installed kubectl using a dedicated setup action so I can apply manifests and check status. I installed Kustomize by downloading a release build and moving it to the PATH. I then installed kind using an action that creates a single-node Kubernetes cluster inside the GitHub Actions runner.

Step 5.3 (Task 5): I validated the cluster by printing cluster information and listing nodes. After that, I added a step that selects the correct overlay path based on the current matrix value: overlays/dev for dev, overlays/staging for staging, and overlays/prod for prod. I ran kustomize build on that directory and saved the output as a single manifest file named after the environment so I can apply it reliably and upload it later as an artifact.

Step 5.4 (Task 5): I applied the generated manifest to the ephemeral cluster using kubectl apply. Because namePrefix changes each object name, I discovered the actual Deployment name dynamically and then ran kubectl rollout status against it with a sensible timeout. When rollout completed successfully, I listed the resources (Deployments, Pods, Services, ConfigMaps, and Secrets) to confirm they exist as expected. Finally, I uploaded the generated manifest as a workflow artifact so I can download and review exactly what Kustomize produced for each environment.

Step 5.5 (Task 5): I connected the local repository to a new GitHub repository, pushed the main branch, and watched the Actions tab. I saw three jobs start automatically, one for each environment, and each job proceeded through checkout, tool installs, cluster creation, build, apply, and rollout verification. With this, the CI/CD integration is complete and observable on every push.

Task 6: Test the CI/CD Pipeline:
To prove the automation responds to configuration edits, I made small, controlled changes and watched the pipeline re-run and re-apply.

Step 6.1 (Task 6): I edited one or more overlay inputs, such as changing the dev welcome message literal or adjusting the staging replica count. I committed and pushed these changes to main, which immediately triggered the workflow again.

Step 6.2 (Task 6): In the Actions view, I opened the latest run and verified that the dev, staging, and prod jobs executed in parallel. For each job, I examined the logs to confirm the sequence: successful checkout, successful tool installation, successful kind cluster setup, successful Kustomize build for the selected overlay, a clean kubectl apply, and a completed kubectl rollout status. I downloaded the artifact produced by each job and confirmed that the updated values appeared in the rendered manifests, indicating that the pipeline correctly rebuilt and redeployed from the modified configuration.

Task 7: Manage Secrets and ConfigMaps:
This task emphasizes correct handling of configuration values and sensitive data through Kustomize generators and shows how the Deployment reads them without hardcoding names that will later be hashed.

Step 7.1 (Task 7): In the base kustomization, I already defined configMapGenerator and secretGenerator to produce app-config and app-secret. In each overlay kustomization, I used behavior: replace so the dev, staging, and prod overlays could provide their own literal values while retaining the same generator names. This pattern keeps naming consistent and keeps values environment-specific without duplicating entire objects.

Step 7.2 (Task 7): In the container spec inside base/deployment.yaml, I added environment variables that pull values from the generated ConfigMap and Secret. Specifically, WELCOME_MESSAGE and FEATURE_FLAG_X come from app-config using configMapKeyRef, and API_TOKEN comes from app-secret using secretKeyRef. The reference names are the logical generator names (app-config and app-secret). Kustomize automatically rewrites these references to the hashed names it generates during the build, which guarantees pods receive the correct, current resources and also ensures that any change to generator content produces a new name and therefore a rolling update.

Step 7.3 (Task 7): I committed and pushed these Deployment changes. During the subsequent workflow run, I confirmed that new hashed names appeared for updated ConfigMaps or Secrets and that rollout completed successfully, demonstrating that Kustomize-driven generators and pod references are functioning as intended.

Task 8: Document Your Work:
This task requires a README that explains how the repository is organized and how someone else can reproduce what I did both locally and in CI.

Step 8.1 (Task 8): I wrote README.md at the project root. In it, I summarized the purpose of the project, described the base and overlays folders, and explained the role of each overlay. I included a short local workflow that installs Kustomize and kind, shows how to create a local cluster, apply an overlay using kustomize build piped to kubectl apply, and then clean up the cluster. I also explained the GitHub Actions process, including triggers, the three-environment matrix, the ephemeral cluster creation, the build and apply steps, the rollout status check, and where to retrieve the generated manifests as artifacts. Lastly, I described how the ConfigMap and Secret generators work and how the Deployment pulls values via valueFrom so that Kustomize can rewrite to hashed names automatically.

Task 9 (Advanced): Implement Transformers and Generators:
This advanced step showcases additional Kustomize features. The repository already demonstrates generators and built-in transformers such as commonLabels, commonAnnotations, and namePrefix. To make the advanced use explicit, I added a label transformer in the production overlay.

Step 9.1 (Task 9): Inside overlays/prod, I created a label transformer that adds a version label to all resources by targeting metadata/labels. This provides clear traceability, especially in production contexts where consistent metadata aids observability and inventory.

Step 9.2 (Task 9): I referenced this transformer in overlays/prod/kustomization.yaml so Kustomize would apply it during the build. After committing and pushing, I inspected the production manifest artifact from the workflow run and confirmed that every object carried the added version label, confirming the transformer worked correctly.

3. Evaluation Criteria:
   I validated the project against the provided criteria. First, Kustomize features are correctly implemented across environments: base contains shared resources and generators, overlays provide name prefixes and patches, and config/secret generators are cleanly overridden with behavior: replace. Second, the CI/CD pipeline is successfully integrated and functional: pushes and pull requests trigger a workflow that installs tools, starts a kind cluster, builds each overlay, applies manifests, verifies rollouts, and uploads the resulting manifests. Third, the approach follows good configuration and secret-management practices: no real secrets are committed, generator hashing is enabled to produce safe rolling updates, and pods reference logical names that Kustomize will rewrite to hashed resource names. Fourth, the documentation is clear and complete: the README provides structure, local usage, and a description of the CI pipeline so another person can reproduce the workflow from scratch.

4. Submission:
   I completed the work, pushed all files to a GitHub repository, and prepared the repository URL for submission. The repository contains the base directory with shared manifests and kustomization, the overlays/dev, overlays/staging, and overlays/prod directories with their kustomization files and patches, the .github/workflows/deploy.yaml workflow, the .gitignore, and the README.md file. This matches the submission requirement to provide the project repository link.

Feedback Request:
I would appreciate any feedback on clarity, structure, and the thoroughness of my explanations. To deepen my understanding while staying within the project’s boundaries, I carefully validated each overlay by rendering manifests, examined how Kustomize rewrites references to hashed names, and re-ran the complete CI/CD flow multiple times to confirm reliable rollouts. Although the screenshots did not require extra experimentation, I took time to study how generator hashes trigger updates and how common labels, annotations, and prefixes simplify multi-environment support. If there are specific naming conventions, additional labels, or verification steps you prefer, please let me know and I will incorporate them.

Conclusion:
This report mirrors the project’s headings and documents, in detail, how I implemented a multi-environment deployment strategy using Kustomize alongside a CI/CD pipeline. The base layer centralizes common resources, while the dev, staging, and prod overlays precisely modify replicas, resource profiles, and configuration values. The pipeline builds, applies, and verifies the manifests automatically for each environment, and the repository includes clear documentation for local and CI use. I have stopped exactly where the project stops, aligning strictly with the screenshots you provided. The images below depict this, numbered exactly as in the VS Code example:

![1img](./1img)
![2img](./2img)
![3img](./3img)
![4img](./4img)
![5img](./5img)
![6img](./6img)
![7img](./7img)
![8img](./8img)
![9img](./9img)
![10img](./10img)
![11img](./11img)
![12img](./12img)
![13img](./13img)
![14img](./14img)
![15img](./15img)
![16img](./16img)

