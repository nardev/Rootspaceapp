# <img width="25" src="https://github.com/nardev/Rootspaceapp/blob/master/web/src/assets/logo.png?raw=true" /> Rootspace app

Rootspace is an innovative and lightweight work management software that offers important features for projects that require online collaboration and task management.  
<br>
It consists of features such as file-sharing, task boards, document editing, and other powerful integrations. Additionally, we have used this platform as a means of encouraging knowledge-sharing and experimentation with emerging technologies among Clearview LLC's team members.  
<br>
One of Rootspace's strongest attributes is its user-friendly design, as well as its ease of deployment and maintenance. The interface is highly intuitive and user-friendly, making it accessible even to the least-experienced team members. It also has the capability to expand its functionality, with new features easily.  
<br>
Rootspace provides a centralized repository for all project information, eliminating the need to search for scattered emails containing URLs, files, access tokens, keys, passwords, and file links, allowing everyone to quickly jump into the project without wasting time searching for information.  

<!-- ABOUT THE PROJECT -->
<h2 id="about-the-project"> :computer: Rootspace runs with</h2>
  
[![Technologies](https://skillicons.dev/icons?i=nodejs,vue,typescript,postgresql,redis,docker)](https://skillicons.dev)  
  
## :eye: A quick preview

![Tasks, Collaboration, Embed, File manager](https://github.com/Rootspaceapp/Rootspace/blob/main/assets/rootspace.gif?raw=true)

> Playing with task lists, Collaborating, Linking external sources, uploading files etc.
    
## :zap: Spin The Rootspace locally
    
After you clone the repo, make a quick copy of .env files and spin the containers defined in docker-compose-try.yml
  
```bash
# navigate to the project's root folder and:
$ cp .env.example .env && cp api/.env.example api/.env && cp web/.env.example web/.env
$ docker compose -f docker-compose-try.yml up
````

 Navigate your browser to:
  > [http://localhost](http://localhost) or [http://localhost:3000](http://localhost:3000)  
  > `# Check your ports. By default the app wiill try to take port 80 and 3000`

 Login with demo user:  
 > username: rootspace@example.com  <br>  password: rootspace  
    
If you need Mailhog, Dozzle or Arena containers, you can also spin those in addition to previous
  
```bash
$ docker compose -f docker-compose-dev.yml up
```
  
  * Access Arena at (use user/pass from api/.env file) http://localhost:3001/arena
  * Access Dozzle to watch container logs in realtime at http://localhost:9999
  * Access sent emails with Mailhog on port http://mailhog:8025

> If required, edit .env files in `/`, `web/`, `api/` folders.

## :scroll: Google, S3 Amazon, Sendgrid API Keys  
  
  In case you want to activate google login, S3 and Sendgrid integration, provide and set required keys in `api/.env`  
  you would need to provide required values and edit api/.env  

| Key      | Description                         | Findout more |
|-------------|------------------------------|-----|
| SENDGRID_API_KEY | Sendgrid email service | [Sendgrid Account Settings](https://docs.sendgrid.com/ui/account-and-settings/api-keys)   |
| S3_ACCESS_KEY | Amazon S3 Access Key  | [AWS Account and Access Keys](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html)  |
| S3_SECRET_KEY | Amazon S3 Secret Key  | [AWS Account and Access Keys](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html)  |
| GOOGLE_CLIENT_ID  | Google Client ID | [Integrating Google Sign-In into your web app](https://developers.google.com/identity/sign-in/web/sign-in)  |
| GOOGLE_CLIENT_SECRET  | Google Client Secret | [Integrating Google Sign-In into your web app](https://developers.google.com/identity/sign-in/web/sign-in)  |

> Other values filled at your discretion, please check api/.env.example
  
## :brain: Debug & Test
> Set `logging: true` in `api/db/db.ts` to see in console all SQL queries sent to PostgreSQL server  

> You can also run `docker-compose up` or `docker-compose up -d` from `tests` directory to bring up ephemeral testing postgres database.

> You can run container-related commands with `infra/cli.sh` script
  
## :card_index_dividers: API docs

You can use [Insomnia](https://insomnia.rest/) and a file `assets/insomnia.json` to check on API. The API container spins at port `3001` by default.
  
## :scroll: License  
![GitHub](https://img.shields.io/github/license/Rootspaceapp/Rootspace)  
  
## :man_technologist: Developed at <img width="22" src="https://github.com/Rootspaceapp/Rootspace/blob/main/assets/cv.jpeg?raw=true" /> [Clearview](https://www.clearview.team)
  
**Team**: Adi Utama, Aditya Purwa, Adnan Puzic, Aid Arslanagic, Arfan Fudyartanto, Jasmin Ihtijarevic, Januar Fonti, Mirza Eka, Mujo Kodro, Nedim Hadzimahmutovic, Taufan Fadhilah Iskandar, Vedran Alajbegovic 
