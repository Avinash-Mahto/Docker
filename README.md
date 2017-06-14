##### Docker
##### Define a docker container and running app through Ansible.

    ---
    - hosts: all
      sudo: yes

      tasks:

      - name: "Install yum utilis package"
        yum: name=yum-utils state=present

      - name: "Add docker-ce repository in yum configuration file"
        command: >
         yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

      - name: "Install docker-ce package"
        yum: name=docker-ce state=present

      - name: "Start docker service"
        service: name=docker state=started

      - name: "Ensure directory is exist"
        file: path=/root/docker state=directory

      - name: "Create empty file with name Dockerfile, requirements.txt and app.py file"
        shell: "{{item}}"
        with_items:
          - touch /root/docker/Dockerfile
          - touch /root/docker/requirements.txt
        tags:
           - directory

      - name: "Insert following lines in Dockerfile"
        copy:
         src: /root/Dockerfile
         dest: /root/docker
        tags:
           - files

      - name: "Insert lines in requirements.txt file"
        shell: "{{ item }}"
        with_items:
          - echo "Flask" > /root/docker/requirements.txt
          - echo "Redis" >> /root/docker/requirements.txt

      - name: "Following lines is isnserted into app.py file"
        copy:
         src: /root/app.py
         dest: /root/docker
        
    - name: Build image and with buildargs
        docker_image:
         path: /root/docker/
         name: friendlyhello
         buildargs:
           log_volume: /var/log/myapp
           listen_port: 4000

      - name: Restart a container
        docker_container:
          name: avinash
          image: friendlyhello
          state: started
          restart: yes
          ports:
           - "4000:80"
##### Here is output of above code:

   ![Capture.JPG](Capture.JPG?raw=true "Title")
   
           
##### After that initialize the swarm service and scale your app

    ---
    - hosts: all
      sudo: yes
      tasks:

      - name: "Initialize swarm cluster"
        shell: >
         docker swarm init --advertise-addr={{ swarm_iface | default('enp0s8') }}:2377

      - name: Copy docker-compose.yml to /root/docker
        copy:
         src: /root/docker-compose.yml
         dest: /root/docker/

      - name: App creation
        command: "docker stack deploy -c /root/docker/docker-compose.yml getstartedlab"
        
##### Need to define following files in your Ansible server at /root
        $ vi Dockerfile


        FROM python:2.7-slim
        WORKDIR /app
        ADD . /app
        RUN pip install -r requirements.txt
        EXPOSE 80
        ENV NAME World
        CMD ["python", "app.py"]

    $ vi app.py

    from flask import Flask
    from redis import Redis, RedisError
    import os
    import socket

    # Connect to Redis
    redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

    app = Flask(__name__)

    @app.route("/")
    def hello():
        try:
            visits = redis.incr("counter")
        except RedisError:
            visits = "<i>cannot connect to Redis, counter disabled</i>"

        html = "<h3>Hello {name}!</h3>" \
               "<b>Hostname:</b> {hostname}<br/>" \
               "<b>Visits:</b> {visits}"
        return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

    if __name__ == "__main__":
            app.run(host='0.0.0.0', port=80)
            

    $ vi docker-compose.yml

    version: "3"
    services:
      web:
        # replace username/repo:tag with your name and image details
        image: friendlyhello
        deploy:
          replicas: 5
          resources:
            limits:
              cpus: "0.1"
              memory: 50M
          restart_policy:
            condition: on-failure
        ports:
          - "80:80"
        networks:
          - webnet
    networks:
      webnet:
      
##### Best of luck
