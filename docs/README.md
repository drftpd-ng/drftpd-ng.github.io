<h2 id="start">Getting started</h2>

Getting started with DrFTPD is quite trivial but to configure it completely takes some love and compassion.

### Requirements

DrFTPD 4.x installation requires a number of steps before you can utilize the software to its full extend.
To give an overview of the installation process the different steps are listed below in this section.

On the master you will need the following applications/tools:
- Java JDK 16 or OpenJDK 16,
- Apache Maven
- Git

On the slaves you will need:
- Java JRE 16 or OpenJDK 16.
- The below are optional dependencies and are only required if you want to use the modules for verifying samples or extracting media info from various filetypes. Optional but recommended.
  - MediaInfo (CLI): https://mediaarea.net/en/MediaInfo
  - mkvalidator tool: https://github.com/Matroska-Org/foundation-source

Now that we covered what is needed, let's move on to cloning the code to the server that will act `master`

?> The following steps expects that you have already installed the above **required** packages

```bash
git clone https://github.com/drftpd-ng/drftpd.git
cd drftpd
git checkout tags/4.0.0
```
