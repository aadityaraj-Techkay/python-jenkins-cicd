# Python Task Manager — AWS EC2 + Jenkins CI/CD

A beginner-friendly Flask project. Jenkins checks out GitHub, installs dependencies, runs a basic test, copies the app to EC2, and restarts it with systemd.

## Architecture
Developer -> GitHub -> Jenkins pipeline -> Ubuntu EC2 -> Flask app on port 5000

> Learning setup: Jenkins and the app run on one EC2 instance to keep the exercise simple. This is not production architecture. AWS may charge for EC2, EBS, public IPv4, data transfer, and other resources. Check Billing and clean up resources when finished.

## 0. Requirements and security group

Use an Ubuntu EC2 instance (guide assumes Ubuntu 22.04/24.04). In its security group allow:
- SSH TCP 22 from your IP only.
- Jenkins TCP 8080 from your IP only (temporary setup access).
- App TCP 5000 from your IP only (learning demo).

Do not expose Jenkins 8080 to 0.0.0.0/0. Check current AWS pricing for your region; do not assume every resource is free.

Connect from your computer:
```bash
chmod 400 ~/Downloads/your-key.pem
ssh -i ~/Downloads/your-key.pem ubuntu@EC2_PUBLIC_IP
```

## 1. Install Java, Python and tools

Run on EC2:
```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre python3 python3-venv python3-pip git rsync wget
java -version
python3 --version
```
If `openjdk-21-jre` is unavailable, check `apt-cache search openjdk | grep jre` and install a Java version supported by your Jenkins release.

## 2. Install Jenkins

For the latest official instructions, see https://www.jenkins.io/doc/book/installing/linux/

Standard Debian/Ubuntu repository commands:
```bash
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

Get initial password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Open `http://EC2_PUBLIC_IP:8080`, paste the password, install suggested plugins, and create your Jenkins admin user. Keep port 8080 restricted to your IP.

## 3. Prepare deployment directory and systemd service

Create a directory Jenkins can write to:
```bash
sudo mkdir -p /opt/python-task-manager
sudo chown -R jenkins:jenkins /opt/python-task-manager
```

From the project folder, copy the service unit to EC2 (or create it there with the content in `python-task-manager.service`):
```bash
sudo cp python-task-manager.service /etc/systemd/system/python-task-manager.service
sudo systemctl daemon-reload
sudo systemctl enable python-task-manager
```

Create the virtual environment as Jenkins:
```bash
sudo -u jenkins python3 -m venv /opt/python-task-manager/venv
sudo -u jenkins /opt/python-task-manager/venv/bin/pip install --upgrade pip
```

Allow Jenkins to restart and check only this service without an interactive password:
```bash
echo 'jenkins ALL=(root) NOPASSWD: /usr/bin/systemctl restart python-task-manager, /usr/bin/systemctl is-active --quiet python-task-manager' | sudo tee /etc/sudoers.d/jenkins-python-task-manager
sudo chmod 440 /etc/sudoers.d/jenkins-python-task-manager
sudo visudo -cf /etc/sudoers.d/jenkins-python-task-manager
```
If `visudo` reports an error, fix it before continuing. Do not grant Jenkins unrestricted passwordless sudo.

## 4. Push project to GitHub

On your own computer, unzip the project:
```bash
cd python-jenkins-cicd
git init
git add .
git commit -m "Add Python task manager and Jenkins pipeline"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/python-task-manager.git
git push -u origin main
```
Create the empty GitHub repository first. Replace the username. For a private repository, configure Jenkins credentials; never put a personal access token in the Jenkinsfile.

## 5. Create Jenkins Pipeline job

1. Jenkins dashboard -> New Item.
2. Name: `python-task-manager-pipeline`.
3. Select Pipeline -> OK.
4. Pipeline section -> Definition: **Pipeline script from SCM**.
5. SCM: Git.
6. Repository URL: `https://github.com/YOUR_GITHUB_USERNAME/python-task-manager.git`.
7. Branch: `*/main`.
8. Script Path: `Jenkinsfile`.
9. Save -> Build Now.
10. Open build number -> Console Output.

Pipeline stages: checkout, install dependencies and run a basic health test, copy files to `/opt/python-task-manager`, restart systemd service.

## 6. Verify app

Open:
- `http://EC2_PUBLIC_IP:5000`
- `http://EC2_PUBLIC_IP:5000/health` (expected JSON: `{"status":"healthy"}`)

On EC2:
```bash
sudo systemctl status python-task-manager --no-pager
sudo journalctl -u python-task-manager -n 50 --no-pager
```

## 7. Test CI/CD

Change a text in `templates/index.html`, then:
```bash
git add .
git commit -m "Update task manager page"
git push
```
Click Build Now in Jenkins. When stages succeed, refresh the app URL and verify the change. GitHub webhook automation can be added later; this first exercise uses Build Now.

## Troubleshooting

**Jenkins page unavailable**
```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -n 80 --no-pager
```
Check inbound TCP 8080 from your current IP.

**App unavailable**
```bash
sudo systemctl status python-task-manager
sudo journalctl -u python-task-manager -n 80 --no-pager
sudo ss -lntp | grep 5000
```
Check inbound TCP 5000 from your current IP.

**Pipeline fails at rsync:** confirm `rsync` is installed and `/opt/python-task-manager` is owned by `jenkins`.

**Pipeline fails at sudo:** run `sudo visudo -cf /etc/sudoers.d/jenkins-python-task-manager`, check `command -v systemctl` (normally `/usr/bin/systemctl`), and verify service name spelling.

**Private GitHub repo clone fails:** add credentials in Jenkins and select them in SCM settings; do not hardcode tokens.

## 8. AWS cleanup

When finished:
```bash
sudo systemctl stop python-task-manager
```
If the lab is no longer needed, terminate the EC2 instance in the console. Check for leftover EBS volumes, snapshots, Elastic IPs and other billable resources. Stopping EC2 does not necessarily stop every associated charge. Review AWS Billing.

## What this project teaches

- Running a Python Flask app on Linux.
- Git/GitHub source control.
- Jenkins jobs, stages and Jenkinsfile.
- Basic CI testing and automated deployment.
- systemd service management and Linux logs.

Note: tasks are stored in memory, not a database, so they reset when the Python process restarts. This is intentional for the first CI/CD exercise.
