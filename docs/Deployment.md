## Requirements

- Node LTS - https://nodejs.org/en/download/
- mongoDB  - https://www.mongodb.com/download-center/community 
- Redis - https://redis.io/download

#### Usage

- Install PM2 (process manager)

```sh
$ sudo npm install -g pm2
```
Note: pm2 is a global install and requires root access 

- Install the app dependencies

```sh
$ npm install
```

- Build and run for development

```sh
$ npm run dev
```

- Build and run for staging

```sh
$ npm run staging
```

- Build and run for production

```sh
$ npm run production
```

- Build Only

```sh
$ npm run build
```

- Build documentations

```sh
$ npm run docs

```

- tail logs

```sh
$ pm2 logs
```

## Environments and ecosystems

The app has 3 different environment settings it can run as: development, staging and production. Each environment setting has it's own set of config files and an ecosystem file.

The ecosystem file is used to start the application under the correct environment and holds a number of overrides vs the config files it used, what ecosystem file you use to start the app determines what config files get used.

More information on ecosystem files can be found at https://pm2.io/doc/en/runtime/guide/ecosystem-file/

#### Examples

Starting application with development ecosystem:

```sh

pm2 start ./ecosystem/development/ecosystem.config.js --env development --update-env

```

where ecosystem.config.js holds following enviroment variables that bootstrap the application on startup


```js
env_development: {
  NODE_ENV: 'development',
  NODE_ENV_HOST: '127.0.0.1',
  NODE_ENV_DOMAIN: '127.0.0.1',
  NODE_ENV_PORT: 8630,
  NODE_ENV_MODE: 'http',
  NODE_ENV_SCHEME: 'http://',
  NODE_ENV_CONFIG_PATH: '/opt/streamovations/straas/configs/api/metadata'
}
```

NODE_ENV is used to determine what mode the application should run in and what config folder it should use (in this case /opt/streamovations/straas/configs/api/metadata/development), when NODE_ENV is set to development debug mode is also enabled, making the application log out non-errors

Note: the command `npm run dev` is a shortcut that does the above + a build for development and is effectively a shortcut for:

```sh

pm2 del ./ecosystem/development/ecosystem.config.js && npm install && npm run build && pm2 start ./ecosystem/development/ecosystem.config.js --env development --update-env && npm audit

```

which does the following:

- delete the app from process manager
- npm install for possible new dependencies
- run gruntfile building the /dist folder from /src
- start the app on process manager using the ecosystem file for development
- run an audit for possible dependencies with known exploits

Both `npm run staging` and `npm run production` do similar things but with a different ecosystem file and environment variables, bootstrapping the application accordingly


## Config

Configuration files are located by default under src/config but can be placed outside the project folder (eg /opt/streamovations) by editing `NODE_ENV_CONFIG_PATH` in the ecosystem files

Each environment setting has its own set of folders: config/development, config/staging and config/production

Configs can be pulled from its own repo at https://gitlab.com/streamovations/straas/configs/api/metadata

```js  
    NODE_ENV_CONFIG_PATH: '/opt/streamovations/straas/configs/api/metadata'
```
## Websockets

Websockets are primarily used by visitors but can be used as server-to-server connection aswell if needed

Connected browsers will join a session room and recieve events for that endpoint when metadata is updated from the cms side, on connection browsers will also receive events with data for attendees, speeches and topics.
All Websocket Data is cached through redis, cache times are configurable in the endpoints config file.

#### Websocket events

Websocket events will for the most part return DB records as data when a CRUD operation is triggered

    - updateDelegate
    - createDelegate
    - deleteDelegate

    - updateMicrophone
    - createMicrophone
    - deleteMicrophone

    - updateDivision
    - createDivision
    - deleteDivision

    - updateRole
    - createRole
    - deleteRole

    - updateSeat
    - createSeat
    - deleteSeat

    - updateAttendee
    - updateAttendeeMicrophone
    - createAttendee
    - deleteAttendee

    - updatePreset
    - createPreset
    - deletePreset
    
    - startSpeech
    - stopSpeech 
    - createSpeech
    - updateSpeech
    - deleteSpeech

    - startTopic
    - stopTopic
    - resetTopic
    - createTopic
    - updateTopic
    - deleteTopic   

#### Example for attendees data

```js
    components.endpoints.forEach(async endpoint => {

        io.of(endpoint).on('connection', async socket => {
            
            let session = socket.handshake.query.session

            if (session && Number.isInteger(session)) {
                
                //send active attendees to browser on connection
                let attendeesCached = await express.redis.getSession(endpoint, 'attendees', session)

                if (attendeesCached) { socket.emit('attendees', attendeesCached) }
                else {

                    let attendees = await express.mongoose.db[endpoint].models.metadata_attendees.find({ session: session, active: true }).populate('delegate').populate('roles')

                    express.redis.cacheSession(endpoint, 'attendees', session, attendees, components.config.endpoints[endpoint].cache.websockets.attendees)
                    
                    socket.emit('attendees', attendees)
                }
                
                //send active attendees to browser when requested
                socket.on('attendees', async data => {

                    let attendeesCached = await express.redis.getSession(endpoint, 'attendees', session)

                    if (attendeesCached) { socket.emit('attendees', attendeesCached) }
                    else {

                        let attendees = await express.mongoose.db[endpoint].models.metadata_attendees.find({ session: session, active: true }).populate('delegate').populate('roles')

                        express.redis.cacheSession(endpoint, 'attendees', session, attendees, components.config.endpoints[endpoint].cache.websockets.attendees)
                        
                        socket.emit('attendees', attendees)
                    }
                })
            }
        })
    })
```
