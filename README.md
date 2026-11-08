
<a name="top"></a>
<h1>Podman Container Management</h1>

### 📋 Tickets Covered in This Repository

- 🐳 [**Ticket #720** — Run the cardealer httpd Container with Podman as a Systemd Service](#ticket-720)

<br />
<br />
<br />
<p align="right"><a href="https://github.com/RexQuaye">🏠 <b>Back to home page</b></a></p>

<br />

---

<br />

<a name="ticket-720"></a>
<h1>Ticket #720 — Run the cardealer httpd Container with Podman as a Systemd Service</h1>

<h2>Languages and Utilities Used</h2>
- <b>Bash</b>
- <b>Podman</b>
- <b>systemd (systemctl --user, loginctl)</b>
- <b>dnf</b>
- <b>ssh</b>
- <b>useradd / passwd</b>
- <b>vim</b>

<h2>Environments Used</h2>
- <b>CentOS Stream 9 (rexq-master-controller)</b>
- <b>Red Hat container registry (registry.redhat.io)</b>
- <b>httpd-24 container image (rhel8/httpd-24)</b>
<br />
<h2>Project walk-through:</h2>

**Task:** Create a container named <code>cardealer</code> using the <code>httpd</code> image available at the registry server <code>registry.redhat.io/rhel8/httpd-24</code>.

**Requirements:**
- Configure it to run as a systemd service that runs for only the user <code>daniel</code>
- Name the service <code>container-cardealer</code> and have it start automatically after a system reboot
- Mount the directory <code>/home/daniel/webcontent/html</code> under <code>/var/www/</code> on the container when it starts, and create a file named <code>index.html</code> in that directory containing "Welcome to Yellow Tail Tech Dealership"
- Mount the directory <code>/home/daniel/car</code> under <code>/var/www/data</code> on the container when it starts, with the <code>/home/daniel/car</code> file containing "Mazda CX5, Ford Taurus"

**Note on redactions:** The password set for daniel and the Red Hat account email used to log in to the registry are redacted with 🤫 emojis. 

<br />

---

<br />

### Install the container tools

- `dnf install container-tools -y` (run as root): installs the container-tools package needed for the Podman task, which pulls in tools such as Buildah, Skopeo and Toolbox. `-y` auto-confirms the prompt.
- **Repository metadata refresh:** dnf downloads current package lists from the CentOS Stream 9 BaseOS, AppStream and Extras repositories.
- **Dependencies resolved:** dnf determines everything container-tools needs and installs it along with the main package.
- **Complete!:** the installation finished successfully.

<p align="center">
<b>rexq-master-controller — Installing the container-tools package:</b>
<br/>
<img width="1318" height="583" alt="dnf install container-tools -y on rexq-master-controller resolving dependencies and finishing with Complete" src="https://github.com/user-attachments/assets/914936ba-54ba-48a8-864a-d4c1e1415856" />




</p>

<br />
<br />
<br />

### Create the user daniel and set a password

- `useradd daniel` (run as root): creates the user account daniel with a home directory at `/home/daniel`. No output means it succeeded.
- `echo <password> | passwd --stdin daniel`: sets daniel's password non-interactively by piping it into passwd, so daniel can log in over SSH for the rootless Podman setup.
- **"all authentication tokens updated successfully":** confirms the password was set.

<p align="center">
<b>rexq-master-controller — Creating the user daniel and setting its password:</b>
<br/>
<b>daniel's password redacted with 🤫 emojis.</b>
<br/>
<img width="1429" height="211" alt="02_create-user-daniel" src="https://github.com/user-attachments/assets/215228f7-aba0-47c6-b021-2eebbc64fa35" />



</p>

<br />
<br />
<br />

### Enable linger for daniel

- `loginctl enable-linger daniel` (run as root): turns on "lingering" for daniel. His user-level systemd services start at boot and keep running after he logs out. Without it, container-cardealer would only run while daniel is logged in.
- **No output:** the command succeeded silently. To confirm, I can run `loginctl show-user daniel | grep Linger`, which should show `Linger=yes`, or `ls /var/lib/systemd/linger`, which should list daniel.
  
<p align="center">
<b>rexq-master-controller — Enabling linger for daniel:</b>
<br/>
<img width="1427" height="155" alt="loginctl enable-linger daniel run as root with no output" src="https://github.com/user-attachments/assets/ef48e8ab-3cb7-427e-926c-758b47f9a7ac" />


</p>

<br />
<br />
<br />

### SSH login as daniel

- `ssh daniel@localhost` (run as root): opens a real login session as daniel on the same machine. A real session is needed so `systemctl --user` works later; `su` wouldn't provide one.
- **Host authenticity prompt:** this is the first SSH connection to localhost, so SSH shows the server's key fingerprint and asks whether to trust it.
- **y then yes:** SSH only accepts the full word. Typing yes confirmed the connection.
- **"Permanently added 'localhost'…":** the host key was saved to known_hosts, so I won't see this prompt again.
- **Password prompt:** I entered daniel's password, which doesn't echo on screen.
- **[daniel@rexq-master-controller ~]$:** I'm now logged in as daniel. The $ prompt means a non-root user.

<p align="center">
<b>rexq-master-controller — Logging in over SSH as daniel:</b>
<br/>
<img width="1418" height="345" alt="04_ssh-as-daniel" src="https://github.com/user-attachments/assets/f7c6be63-e4f0-4c9a-9b68-d58e3f373d3e" />


</p>

<br />
<br />
<br />

### Log in to the Red Hat registry

- `podman login registry.redhat.io` (as daniel): logs in to the Red Hat registry so the httpd image can be pulled.
- **Username and Password prompts:** the username is the Red Hat account used for the registry, and the password isn't echoed on screen.
- **Login Succeeded!:** confirms the login worked.

<p align="center">
<b>rexq-master-controller (as daniel) — Logging in to registry.redhat.io:</b>
<br/>
<b>Redaction: the Red Hat account email is covered with 🤫 emojis.</b>
<br/>
<img width="1190" height="139" alt="05_registry-login" src="https://github.com/user-attachments/assets/9506da0c-f7f8-4583-bcf1-54941a7e2ea6" />


</p>

<br />
<br />
<br />

### Pull and verify the httpd-24 image

- `podman pull registry.redhat.io/rhel8/httpd-24` (as daniel): downloads the httpd image into daniel's rootless image store. No tag was given, so Podman pulled `:latest`.
- **"Getting image source signatures" / "Checking if image destination supports signatures":** Podman checks the image's signatures from Red Hat.
- **"Copying blob … done" (3 lines):** the image's layers are downloaded.
- **"Copying config … done" / "Writing manifest":** the image's configuration and metadata are saved locally.
- **"Storing signatures" and the long hash:** the pull finished, and the hash is the full image ID.
- `podman images`: lists daniel's local images. It confirms registry.redhat.io/rhel8/httpd-24, tag latest, image ID 2d93630ebe11, 295 MB.

<p align="center">
<b>rexq-master-controller (as daniel) — Pulling the httpd-24 image and verifying it with podman images:</b>
<br/>
<img width="1218" height="358" alt="06_pull-httpd-image" src="https://github.com/user-attachments/assets/3f347118-68bd-4ee7-aa9d-7beb92be4e4d" />


</p>

<br />
<br />
<br />

### Create the web page index.html

- `vim /home/daniel/webcontent/html/index.html` (as daniel): opens or creates the web page file in vim.
- **Text line:** the page content, typed in insert mode.
- **~ lines:** empty lines below the end of the file, which vim marks with ~. They are not part of the file.
- **:wq:** writes (saves) the file and quits vim. The prompt came back, so the save worked.

<p align="center">
<b>rexq-master-controller (as daniel) — Creating index.html in vim:</b>
<br/>
<img width="1187" height="271" alt="07_vim-index-html" src="https://github.com/user-attachments/assets/90edeb83-17fb-48e8-9b2a-31d4401250ce" />


</p>

**Then verify:** `cat /home/daniel/webcontent/html/index.html` prints the page content.

<p align="center">
<b>rexq-master-controller (as daniel) — Verifying the contents of index.html:</b>
<br/>
<img width="1182" height="131" alt="08_verify-index-html" src="https://github.com/user-attachments/assets/6bab08dc-1571-48d0-bc6f-163c9e91b608" />


</p>

<br />
<br />
<br />

### Create the car data file

- `vim /home/daniel/car` (as daniel): creates and opens the car file in vim.
- **Mazda CX5, Ford Taurus:** the file's content, typed in insert mode without quotes. This is correct.
- **~ lines:** empty space past the end of the file. They are not part of the file.
- **:wq:** saves the file and quits vim. The prompt came back, so the save worked.

<p align="center">
<b>rexq-master-controller (as daniel) — Creating the car file in vim:</b>
<br/><img width="1186" height="240" alt="09_vim-car-file" src="https://github.com/user-attachments/assets/84695abb-267b-4b17-8075-77dbc5d179ec" />


</p>

**Then verify:** `cat /home/daniel/car` prints the file content.

<p align="center">
<b>rexq-master-controller (as daniel) — Verifying the contents of the car file:</b>
<br/>
<img width="1181" height="143" alt="10_verify-car-file" src="https://github.com/user-attachments/assets/dcc21031-2bfd-4f9b-a3e7-81058d8dab45" />


</p>

<br />
<br />
<br />

### Run the cardealer container with volume mounts

- `podman run -d`: creates and starts a container in the background (detached).
- `--name cardealer`: names the container cardealer.
- `-v /home/daniel/webcontent/html:/var/www/html:Z`: mounts the web content folder so httpd serves my index.html.
- `-v /home/daniel/car:/var/www/data:Z`: mounts the car file at `/var/www/data` inside the container.
- `:Z`: relabels the files for SELinux so the container is allowed to read them.
- `registry.redhat.io/rhel8/httpd-24`: the image to run.

<p align="center">
<b>rexq-master-controller (as daniel) — Running the cardealer container with both volume mounts:</b>
<br/>
<img width="1188" height="168" alt="11_run-cardealer" src="https://github.com/user-attachments/assets/b4b3eeff-c497-46f5-8a62-93f4727db64a" />


</p>

<br />
<br />
<br />

### Verify the container is running

- `podman ps` (as daniel): lists daniel's running containers.
- **CONTAINER ID 850594d4eed8:** a short form of the ID printed by podman run.
- **IMAGE:** `registry.redhat.io/rhel8/httpd-24:latest`, the image the container runs.
- **COMMAND `/usr/bin/run-http…`:** the image's startup script, which launches Apache httpd.
- **STATUS "Up 3 minutes":** the container is running.
- **PORTS 8080/tcp, 8443/tcp:** the ports httpd listens on inside the container. They aren't published to the host, and the objective doesn't ask for that.
- **NAMES cardealer:** confirms the container name.

<p align="center">
<b>rexq-master-controller (as daniel) — Verifying cardealer is running with podman ps:</b>
<br/>
<img width="1206" height="248" alt="12_verify-running" src="https://github.com/user-attachments/assets/e482192c-65ca-42f8-900a-4413b984c31e" />


</p>

<br />
<br />
<br />

### Verify the mounts from inside the container

- `podman exec -it cardealer /bin/bash`: opens a shell inside the container.
- **bash-4.4$:** the container's shell prompt. I'm now inside the container, not on the host.
- `cat /var/www/html/index.html`: shows the web page content through the first mount.
- `cat /var/www/data`: shows the car data through the second mount.
- `exit`: closes the container's shell and returns me to the host. The container keeps running.
- **[daniel@rexq-master-controller ~]$:** I'm back on the host as daniel.

<p align="center">
<b>rexq-master-controller (as daniel) — Verifying both mounts from inside the container:</b>
<br/>
<img width="1203" height="318" alt="13_verify-mounts" src="https://github.com/user-attachments/assets/94830738-d748-442e-892b-604d1fd9d686" />


</p>

<br />
<br />
<br />

### Generate the container-cardealer systemd service file

- `mkdir -p ~/.config/systemd/user`: creates the folder where daniel's personal systemd service files live. `-p` also creates any missing parent folders.
- `cd ~/.config/systemd/user`: moves into that folder. The prompt changes from `~]$` to `user]$`.
- `podman generate systemd --name cardealer --files --new`: creates a systemd unit file from the cardealer container.
  - `--name`: uses the container's name, so the file is called container-cardealer.service.
  - `--files`: writes the unit to a file instead of printing it on screen.
  - `--new`: the service creates a fresh container each time it starts, which is more reliable after a reboot.
- **DEPRECATED message:** a notice that newer setups use Quadlet. It's safe to ignore.
- `/home/daniel/.config/systemd/user/container-cardealer.service`: confirms the file was created with the required name.

<p align="center">
<b>rexq-master-controller (as daniel) — Generating the container-cardealer systemd service file:</b>
<br/>
<img width="1182" height="348" alt="14_generate-service-file" src="https://github.com/user-attachments/assets/a65b5690-876f-423f-b986-1272d1656ba4" />


</p>

<br />
<br />
<br />

### Stop and remove the manual cardealer container

- `podman stop cardealer`: stops the manually started container. Printing cardealer confirms it stopped.
- `podman rm cardealer`: deletes the stopped container so the systemd service can create a fresh one with the same name. Printing cardealer confirms it was removed. The image and mounted files are not affected.

<p align="center">
<b>rexq-master-controller (as daniel) — Stopping and removing the manual cardealer container:</b>
<br/>
<img width="1188" height="133" alt="15_stop-remove-manual" src="https://github.com/user-attachments/assets/1dcee02c-e658-4749-9171-0f4f392e5f09" />


</p>

<br />
<br />
<br />

### Enable and start the container-cardealer service

- `systemctl --user daemon-reload`: makes daniel's systemd re-read its unit files so it sees the new container-cardealer.service. No output means it succeeded.
- `systemctl --user enable --now container-cardealer.service`:
  - `enable`: makes the service start automatically when daniel's systemd starts. With linger on, that happens at boot.
  - `--now`: also starts the service immediately.
- **"Created symlink … default.target.wants/container-cardealer.service":** confirms the service is enabled. The link tells systemd to start it automatically.

<p align="center">
<b>rexq-master-controller (as daniel) — Enabling and starting container-cardealer.service:</b>
<br/><img width="1089" height="177" alt="16_enable-start-service" src="https://github.com/user-attachments/assets/89aa30bc-443f-496e-a555-3dd6319f83ea" />


</p>

<br />
<br />
<br />

### Verify cardealer is running as a systemd service

- `podman ps`: lists daniel's running containers.
- **CONTAINER ID 0fafa2556ca5:** a new ID, different from the earlier 850594d4eed8. This shows systemd created this container, not my earlier manual podman run.
- **STATUS "Up 2 minutes":** the container is running under the service.
- **NAMES cardealer:** the name is correct.

<p align="center">
<b>rexq-master-controller (as daniel) — Verifying cardealer is running under systemd:</b>
<br/><img width="1162" height="171" alt="17_verify-systemd-running" src="https://github.com/user-attachments/assets/4abb3a6c-a3a7-48a7-815e-d859052f66f6" />


</p>

<br />
<br />
<br />

<b><u>Solution:</u></b> I installed the container tools on rexq-master-controller, created the user daniel, and enabled linger so daniel's user services can run without an active login. Logged in over SSH as daniel, I signed in to registry.redhat.io and pulled the httpd-24 image into daniel's rootless image store. I created <code>index.html</code> ("Welcome to Yellow Tail Tech Dealership") in <code>/home/daniel/webcontent/html</code> and a car file containing "Mazda CX5, Ford Taurus" in <code>/home/daniel/car</code>. I then ran the container named <code>cardealer</code> with both locations mounted, the web content at <code>/var/www/html</code> and the car file at <code>/var/www/data</code>, with the <code>:Z</code> option for SELinux, and confirmed from inside the container that both showed the right contents. I generated the systemd unit with <code>podman generate systemd --name cardealer --files --new</code>, which created <code>container-cardealer.service</code> in daniel's <code>~/.config/systemd/user</code> folder, removed the manual container, then enabled and started the service. <code>podman ps</code> afterwards showed cardealer running with a new container ID, which confirms systemd created it. Because it is a user service for daniel only and linger is enabled, the service is set to start at boot.

<br />
<br />

<p align="right"><a href="#top">⬆ Back to top</a></p>

<br />
<br />
