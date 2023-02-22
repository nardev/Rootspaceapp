# <img width="25" src="https://github.com/nardev/Rootspaceapp/blob/master/web/src/assets/logo.png?raw=true" /> Rootspace app

Rootspace is an innovative and lightweight work management software that offers important features for projects that require online colaboration and task management.  
<br>
It consists of features such as file-sharing, task boards, document editing, and other powerful integrations. Additionally, we have used this platform as a means of encouraging knowledge-sharing and experimentation with emerging technologies among Clearview LLC's team members.  
<br>
One of Rootspace's strongest attributes is its user-friendly design, as well as its ease of deployment and maintenance. The interface is highly intuitive and user-friendly, making it accessible even to the least experienced team members. It also has the capability to expand its functionality with new features easily.  
<br>
Rootspace provides a centralized repository for all project information, eliminating the need to search for scattered emails containing URLs, files, access tokens, keys, passwords, and file links, allowing everyone to quickly jump into the project without wasting time searching for information.  

<!-- ABOUT THE PROJECT -->
<h2 id="about-the-project"> :computer: Rootspace runs with</h2>
<br>  
  
[![My Skills](https://skillicons.dev/icons?i=nodejs,vue,typescript,postgresql,redis,docker)](https://skillicons.dev)  
<br>  

## :zap: Spin Rootspace locally
<br>  
Just essential containers:
    
```bash
$ docker compose up -d
````
  
  * Navigate your brosers to: [http://localhost:3000](http://localhost:3000)  
  
<br>  

If you need Mailhog, Dozzle or Arena caontainers, you can also spin those
```bash
$ docker compose -f docker-compose-dev.yml
```
  * Access Arena at (use user/pass from api/.env file) http://api:3001/arena
  * Access Dozzle to watch container logs in realtime at http://localhost:9999
  * Access sent emails with Mailhog on port http://mailhog:8025

> If required, edit .env files in `/`, `web/`, `api/` folders.

> Quick run of typeorm with script `infra/typeorm.sh + [operation]`  

> Note: You can also run `docker-compose up` or `docker-compose up -d` from `tests` directory to bring up ephemeral testing postgres database.

## :cloud: Deployment:

  In case you want to activate google login, S3 and Sendgrid integration, provide and set required keys in `api/.env`

> You can run container-related commands with `infra/cli.sh` script

## Debug
Set `logging: true` in `api/db/db.ts` to see in console all SQL queries sent to PostgreSQL server

## :man_technologist: Authors

* Adi Utama :indonesia:
* Aditya Purwa :indonesia:
* Adnan Puzic :bosnia_herzegovina:
* Aid Arslanagic :bosnia_herzegovina:
* Arfan Fudyartanto :indonesia:
* Jasmin Ihtijarevic :bosnia_herzegovina:
* Januar Fonti :indonesia:
* Mirza Eka :bosnia_herzegovina:
* Mujo Kodro :bosnia_herzegovina:
* Nedim Hadzimahmutovic :bosnia_herzegovina:
* Taufan Fadhila :indonesia:

## :scroll: License  
![GitHub](https://img.shields.io/github/license/Rootspaceapp/Rootspace)  
  
## API docs

You can use Insomnia file to test API `assets/insomnia.json`. The API container spins at port `3001` by default.
