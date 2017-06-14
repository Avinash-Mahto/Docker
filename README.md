##### Docker
##### Define a docker container and running app through it.

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
