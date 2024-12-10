# Probes ( Liveness, Readyness and Startup )

## Liveness probe

- Liveness probe determine when to restart a container, For example, <u>liveness probe could catch a deadlock when an application is running but unable to make progress </u>.
- If a container failes its liveness probe repeatedly, the kubelet restart the container
- <u> Liveness probe do not wait for Readyness probe to be succeded </u>. If you want to wait before executing the liveness probe, you can either define **initialDelaySeconds** or use a startup probe.

  ### Example

  ```
  livenessProbe:
      exec:
      command:
          - cat
          - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
  ```

- Livenessprobe to check the HTTP endpoint

  ```
  livenessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 30
    periodSeconds: 30
  ```

- Liveness probe to check the TCP soecket

  ```
  livenessProbe
    tcpSocket:
      port: 80
    initialDelaySeconds: 30
    periodSeconds: 30
  ```

- In the above example the pod has a single Container.periodSeconds field specifies that kubelet should perform a liveness probe every 5 secods.
  The initialDelaySeconds field tells the kubelet that it should wait 5 seconds before performing the first probe.

## Readiness Probe

- <u>Readyness probe determine when a container is ready to accept the traffic.</u>
- This is useful when waiting for an application to perform time consuming initial tasks, such as establishing network connections, loading files and warming caches.
- <u>If the readiness probe returns a filed state, Kubernetes removes the pod from all matching service endpoints.</u>
- Readyness probe run on the container during its whole lifecycle.
  ### Example
- ReadinessProbe to check the input file is present or not
  ```
  readinessProbe:
    exec:
      command:
        - cat
        - /tmp/data/input.json
    initialDelaySeconds: 5
    periodSeconds: 5
  ```

## Startup Probe

- <u> A startup probe verifies whether the application within a container is started.</u>
- This can be used to adopt liveness checks on slow starting containers, avoiding them getting killed by the kubelet before they are up and running.
- If such probe is configures, it disables readiness and liveness checks until it succeeds.
- The startup probe only executes at the container startup, unlike the readiness and liveness probes run periodically.
  ### Example
- Startup probe to check the input data file is present or not
  ```
  startupProbe:
    exec:
      command:
        - cat
        - /tmp/data/input.json
    initialDelaySeconds: 30
    periodSeconds: 30
  ```
