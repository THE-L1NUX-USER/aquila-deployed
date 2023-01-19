# Some challanges faced while deploying the application.

## Challange 1

While on the initial setup page in `Connection string to MongoDB`
wasn't able to connect to the database using the default entry:
```
 mongodb://localhost:27017/database
```

This was fixed by using the IP address of the docker container of mongo.

Instead of using localhost I used the IP address of the `mongo` docker container.

```
 mongodb://172.18.0.2:27017/database
```
Which resulted in sucessfull connection to database.

To find the IP address of the container `mongo`
```
docker inspect mongo | grep IP
```

### Furthermore,

I remembered in a same netwrok containers can talk to each other by their name only as they can resolve IP address from `container name`.

So, I changed the IP adress to the container name:

```
mongodb://mongo:27017/database
```

Which also resulted in successful connection to the database.

## Challange 2

After filling the `Adminstrative details` in the setup and clicking on `Save Configuration`.

I was greated with another error:

![](https://media.discordapp.net/attachments/853458017738555403/1065707424620748862/image.png)

`A Gateway Time-Out`

Which occours only if there is too much traffic or the server is not sending any data.

As there wasnt any traffic on the website.
The error should be from the application side as it not sending the data.
Or the data is being `blocked`.

Then I checked if i was connected to the vpn or not.

I was not.

As the application is sending data to the admin page which is going to be `blocked` as it won't find the `VPN` IP.

After connection to `VPN` the apllication setup ran successfully.
