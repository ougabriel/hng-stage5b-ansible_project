# hng-stage5b-ansible_project



This documentation provides the steps to set up nestjs on a server using an Ansible playbook. The playbook will read environment variables from a `.env.local` file and use these variables to configure PostgreSQL.

#### Prerequisites

- Ansible installed on your control node.
- Target servers (`webserver1` and `dbserver1`) with SSH access configured.
- `.env.local` file with the necessary environment variables.

#### Directory Structure

Project directory should be organized as follows:

```
ansible_project/
├── inventory.cfg
├── main.yaml
├── roles/
│   ├── webserver/
│   │   ├── tasks/
│   │   │   └── main.yaml
│   │   ├── vars/
│   │   │   └── vars.yml
│   ├── database/
│   │   ├── tasks/
│   │   │   └── main.yaml
```

#### File Contents

1. **`inventory.cfg`**: Define your inventory file with the target servers.

    ```ini
    [webservers]
    webserver1 ansible_host=your.webserver1.ip

    [dbservers]
    dbserver1 ansible_host=your.dbserver1.ip
    ```

2. **`main.yaml`**: Define the main playbook.

    ```yaml
    ---
    - name: Setup Web Server
      hosts: webservers
      roles:
        - webserver

    - name: Setup Database Server
      hosts: dbservers
      roles:
        - database
    ```

3. **`roles/webserver/tasks/main.yaml`**: Define tasks for the web server role.

    ```yaml
    ---
    - name: Update and upgrade apt packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install required packages
      apt:
        name:
          - python3-psycopg2
          - postgresql
        state: present

    - name: Set Python interpreter
      set_fact:
        ansible_python_interpreter: /usr/bin/python3

    - name: Save PostgreSQL credentials
      copy:
        dest: /var/secrets/pg_pw.txt
        content: |
          username: "{{ pg_username }}"
          password: "{{ pg_password }}"
        owner: root
        group: root
        mode: '0600'
    ```

4. **`roles/webserver/vars/vars.yml`**: Define variables for the web server role.

    ```yaml
    pg_username: username
    pg_password: password
    ```

5. **`roles/database/tasks/main.yaml`**: Define tasks for the database server role.

    ```yaml
    ---
    - name: Include webserver role vars
      include_vars:
        file: ../webserver/vars/vars.yml

    - name: Install required packages
      apt:
        name:
          - python3-psycopg2
          - postgresql
        state: present

    - name: Set Python interpreter
      set_fact:
        ansible_python_interpreter: /usr/bin/python3

    - name: Ensure PostgreSQL is running
      service:
        name: postgresql
        state: started
        enabled: yes

    - name: Backup pg_hba.conf
      copy:
        src: /etc/postgresql/14/main/pg_hba.conf
        dest: /etc/postgresql/14/main/pg_hba.conf.bak
        remote_src: yes

    - name: Change local authentication to md5
      lineinfile:
        path: /etc/postgresql/14/main/pg_hba.conf
        regexp: '^local\s+all\s+postgres\s+peer'
        line: 'local   all             postgres                                md5'
        state: present

    - name: Change host authentication to md5
      lineinfile:
        path: /etc/postgresql/14/main/pg_hba.conf
        regexp: '^host\s+all\s+all\s+127.0.0.1/32\s+trust'
        line: 'host    all             all             127.0.0.1/32            md5'
        state: present

    - name: Change IPv6 host authentication to md5
      lineinfile:
        path: /etc/postgresql/14/main/pg_hba.conf
        regexp: '^host\s+all\s+all\s+::1/128\s+trust'
        line: 'host    all             all             ::1/128                 md5'
        state: present

    - name: Restart PostgreSQL to apply changes
      service:
        name: postgresql
        state: restarted

    - name: Read environment variables from .env.local
      shell: |
        set -a
        source /opt/stage_5b/.env.local
        echo "$DB_USERNAME"
        echo "$DB_PASSWORD"
        echo "$DB_DATABASE"
        echo "$DB_HOST"
        echo "$DB_PORT"
        set +a
      register: env_vars

    - name: Debug environment variables
      debug:
        msg: |
          DB_USERNAME: {{ env_vars.stdout_lines[0] }}
          DB_PASSWORD: {{ env_vars.stdout_lines[1] }}
          DB_DATABASE: {{ env_vars.stdout_lines[2] }}
          DB_HOST: {{ env_vars.stdout_lines[3] }}
          DB_PORT: {{ env_vars.stdout_lines[4] }}

    - name: Create PostgreSQL user
      postgresql_user:
        name: "{{ env_vars.stdout_lines[0] }}"  # DB_USERNAME
        password: "{{ env_vars.stdout_lines[1] }}"  # DB_PASSWORD
        state: present
        login_user: postgres
        login_password: "{{ env_vars.stdout_lines[1] }}"  # DB_PASSWORD

    - name: Create PostgreSQL database
      postgresql_db:
        name: "{{ env_vars.stdout_lines[2] }}"  # DB_DATABASE
        owner: "{{ env_vars.stdout_lines[0] }}"  # DB_USERNAME
        state: present
        login_user: postgres
        login_password: "{{ env_vars.stdout_lines[1] }}"  # DB_PASSWORD
        encoding: UTF-8

    - name: Grant all privileges on database to user
      postgresql_privs:
        db: "{{ env_vars.stdout_lines[2] }}"  # DB_DATABASE
        role: "{{ env_vars.stdout_lines[0] }}"  # DB_USERNAME
        privs: ALL
        type: database
        login_user: postgres
        login_password: "{{ env_vars.stdout_lines[1] }}"  # DB_PASSWORD
    ```

#### Running the Playbook

Run the playbook with the following command:

```bash
ansible-playbook main.yaml -i inventory.cfg -b
```

This command will execute the tasks defined in the playbook, set up the web server and database server, and configure PostgreSQL using the environment variables from the `.env.local` file.

### Notes

- Ensure the `.env.local` file is correctly placed on the target server (`dbserver1`) at `/opt/stage_5b/.env.local`.
- Ensure the environment variables in `.env.local` are correctly defined (e.g., `DB_USERNAME`, `DB_PASSWORD`, `DB_DATABASE`, `DB_HOST`, `DB_PORT`).

Thanks for reading, refer to repo dir to access the `webserver` script
