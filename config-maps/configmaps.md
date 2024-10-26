# Kubernetes ConfigMaps
- Creating config maps
  1. Using the kubectl commandline tool   
    `kubectl create configmap app-config --from-literal=PORT=5050 --from-literal=API_ENDPOINT=http://localhost:8000`
  2. Using a config file with key value pairs   
  `kubectl create configmap --from-file=/path/to/file.properties`
  3. Using a manifest file   
  `kubectl apply -f /path/to/manifest.yaml`   
  `kubectl create -f /path/to/manifest.yaml`
- Deleting a configmap
  1. Using the kubectl commandline   
  `kubectl delete configmap/<config_map_name>`
  - Using a configmap in pod definition manifest
    - mapping a field from ConfigMap to environment variable of a pod
      ```
      containers:
        - name: sample-container
          image: <image_name>
          environment:
            - Name: PORT
              valueFrom:
                configMapKeyRef:
                  Key: <config_map_name>
                  Value: PORT
      ```
    - mapping an entire configmap as a environment in a pod
      ```
      containers:
        - name: sample-container
          image: <image_name>
          envFrom:
            - configMapRef:
                name: <config-map-name>
      ```
  - mapping a config map as a volume mount in the pod
    ```
    containers:
        - name: sample-container
          image: <image_name>
          volumeMounts:
            - name: config-volume
              mountPath: </path/volume>
    volumes:
        - name: config-volume
          configMap:
            name: <config_map_name>    
    ```