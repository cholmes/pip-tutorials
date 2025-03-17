# Getting PV data from Planet with CLI tools
## About
This document is written to give step by step directions for how to generate and download Planetary Variable Data from Planet Insights Platform, for AAFC. It mostly uses ‘command-line interface’ tools that you use in your terminal. So it should not require Python knowledge, but does involve using the command-line (‘terminal’ in mac or linux, ‘windows terminal’ in windows). 
## Set-Up
Though no python programming is needed you do need your terminal set up with python, as there are two python packages that need to be installed. Once you’ve got python installed on the command line you just need to run two commands:

```
pip install planet
pip install sh-downloader
```

Then you’ll need to configure both with your appropriate API keys.  First Planet:

`planet auth init`

Just type that on the console, and it should prompt you for your email and password for your Planet account, and then you’ll be set up. If it doesn’t prompt you for email then it likely was not installed correctly. We’ll set up the sh-downloader later.

The following shows roughly how it should look (though I already have the library set up, so it raised some errors)

![planet-cli-install](https://github.com/user-attachments/assets/c8b49d6b-672e-4ca1-8cf6-0b29c16cc135)

## Uploading AOI and Reserving Quota
Next you’ll need to upload your AOI and reserve your quota so you can place a subscription. For now we’ll just focus on creating a single AOI, making a subscription on that and downloading the results. But in a next tutorial we can show uploading a file with many geometries. You can find the interface at https://planet.com/features The process looks like this:


![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/features-manager.gif)

## Creating the Subscription
Now that you’ve reserved the quota you’re ready to create the subscription. For this you need to get a reference to the geometry you created. Just click on the 3 vertical dots by the feature, and then select ‘copy reference’. 
![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/Screenshot%202025-03-13%20at%204.08.34%20PM.png)
Then you can paste it, it should look something like `pl:features/my/alberta-aoi-wr3oRyr/1-ober8nm`.

Then you can take that id as your geometry. The command will look something like this:

`planet subscriptions request-pv --var-type soil_water_content --var-id SWC-SMAP-L_V2.0_100 --geometry pl:features/my/alberta-aoi-wr3oRyr/1-ober8nm --start-time "2024-01-01" --end-time "2024-09-01" > pv-request.json`  

The var-type is the type of Planetary Variable., and the id the specific data product. The table below can help (and the links to go the product specifications that have more details.

| **PV**                                                       | **var-type**       | var-ids                                                      |
|--------------------------------------------------------------|--------------------|--------------------------------------------------------------|
| [Soil Water Content](https://developers.planet.com/docs/planetary-variables/soil-water-content-technical-specification/) | soil_water_content | SWC-SMAP-L_V2.0_100, SWC-AMSR2-C_V2.0_100, SWC-AMSR2-X_V2.0_100, SWC-SMAP-L_V5.0_1000 |
| [Crop Biomass](https://developers.planet.com/docs/planetary-variables/crop-biomass-technical-specification/) | biomass_proxy      | BIOMASS-PROXY_V4.0_10                                        |

The start time and end time are hopefully self-explanatory - the date range for where you want data produced. If you leave off the end time then it will be an ongoing subscription, adding new data every day.

The call above will create a file called `pv-request.json`, which you’ll use in the next step.

`planet subscriptions request --name my_subscription_52 --source pv-request.json --hosting sentinel_hub --create-configuration > full-request.json`

This creates a file called `full-request.json`. You should set your own name (you can use a name with spaces, just put it in quotes like “Subscription 52”. This will automatically use your planet account to deliver to Sentinel Hub, and `—create-configuration` ensures that it can be visualized and used right away.

Then finally you take that request and create the subscription:

`planet subscriptions create full-request.json`

![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/cli-order-sub.gif)

You can also combine all three requests into a single line, instead of creating files the pipe (`|`) and dash ( `-`) pass the output from each into the next one:

``` 
% planet subscriptions request-pv --var-type soil_water_content --var-id SWC-SMAP-L_V2.0_100 --geometry pl:features/my/lethbridge-Yvw3VVv/1-mEeXVlm --start-time "2024-01-01" | planet subscriptions request --name Singg_Alberta_Ref --source - --hosting sentinel_hub --create-configuration | planet subscriptions create -
```

You should get a long string of JSON text back that has the information on the created subscription, like `{"name": "My Sub 55", "source": {"type": "soil_water_content", "parameters": {"id": "SWC-SMAP-L_V2.0_100", "geometry": {"type": "Polygon", "coordinates": [[[-112.77910159, 49.66882007], [-112.62240674, 49.66853052], [-112.62325894, 49.75598645], [-112.77877446, 49.75623658], [-112.77910159, 49.66882007]]]}, "geom_ref": "pl:features/my/alberta-aoi-wr3oRyr/1-ober8nm", "start_time": "2024-01-01T00:00:00Z", "end_time": "2024-09-01T00:00:00Z"}}, "hosting": {"type": "sentinel_hub", "parameters": {"collection_id": "b7435e70-691b-4302-b2a8-a1a5cf030c6a", "configuration_id": "2d9e48ee-fca9-49ba-90bd-5fba3e1a956c"}}, "created": "2025-03-13T23:26:50.95569Z", "_links": {"_self": "https://api.planet.com/subscriptions/v1/06b6261d-1d2c-4040-8cbf-cb83e32aad9b", "results": "https://api.planet.com/subscriptions/v1/06b6261d-1d2c-4040-8cbf-cb83e32aad9b/results"}, "status": "preparing", "id": "06b6261d-1d2c-4040-8cbf-cb83e32aad9b", "updated": "2025-03-13T23:26:50.95569Z"}`

You can also type `planet subscriptions list —limit 1 | jq` to get the same info (and if you have ‘jq’ installed it’ll print it nicely formatted. If you don’t have it just leave off `| jq`

You should also be able to confirm your subscription was created by going to [Planet Insights Platform Subscription Page](https://insights.planet.com/data/subscriptions). There you should see it listed, likely in ‘preparing’.

![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/Screenshot%202025-03-13%20at%204.31.31%20PM.png)

## Getting your data
The next step is to wait. It can take a few hours for your subscription to run. If the status is 'pending' then it's not yet started, and is waiting for compute resources. If it says 'running' then data is being generated. You may be able to download some results during this time. If it says 'completed' then the data is definitely ready to download. Ongoing subscriptions will not say 'completed', but will still say running. 

From the Subscriptions UI you can follow the link to visualize the collection (once there’s results). And you can use the Browser app to check on the data and also do basic time series analysis on the pixels of the time series.

![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/subscriptions-pip2.gif)

### Using sh-downloader
To get your data off of Planet Insights Platform you can make use of a tool we just developed. It might be a little bit buggy, so if you have problems don’t hesitate to reach out. 

You install it similarly to the planet cli:

`pip install sh-downloader`

You can check if it’s working by typing `shdown —help` and it should look something like:![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/Screenshot%202025-03-13%20at%204.45.52%20PM.png)

Next you need to configure it. You need a Sentinel Hub Client ID and Client Secret. Go to https://apps.sentinel-hub.com/dashboard/#/account/settings and hit '+ Create' under OAuth Clients. Be sure to store these, you won't see 'secret' again. And then you put those into the prompt. And you can get the instance id from [The Configurations page](https://insights.planet.com/analyze/configurations/#/) - it’s just the ‘configuration id’.

![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/shdown-config.gif)

Then to get your data down you just use the `byoc` call. Just get your collection id (same as the instance id you configured) that you want to download, and you should be able to just type `shdown byoc <your-id>`

![](Getting%20PV%20data%20from%20Planet%20with%20CLI%20tools/shdown-download.gif)

For this type of PV data there are some options to make the data more closely match the data that gets delivered to a cloud bucket, namely setting the nodata and scale values correctly. You can also select which bands you want, like just get the SWC band.

(TODO: Add a bit more on the options)

