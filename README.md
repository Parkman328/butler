# Butler for Qlik Sense

Node.js based proxy app for providing features accessible from Qlik Sense load scripts (or other systems capable of making REST calls).

The app started out as a way of posting to [Slack](https://slack.com/) from [Qlik Sense](http://www.qlik.com/products/qlik-sense) (or [QlikView](http://www.qlik.com/products/qlikview)) load scripts, but has since been generalized and now supports the following features/endpoints:

- Post to Slack.
      Endpoint: /slack
- Create directories on the server where slack_proxy is running
      Endpoint: /createDir
- Check available disk space
      Endpoint: /getDiskSpace
- Start Qlik Sense tasks based on MQTT subscriptions 
      Endpoint: /mqttStartSenseTask
- Publish message to MQTT topic
      Endpoint: /mqttPublishMessage 

Other endpoints can be added if/when needed - ideas include linking to services like [Pushover](https://pushover.net/), sending tweets, controlling USB status lights like [Blink(1)](https://blink1.thingm.com/) etc. Given the large number of node.js modules available, it is quite easy to add new integrations.


Configuration
-------------
The following configuration needs to be entered in the source code:
- Enter the webhook URL you got when configuring incoming webhooks in Slack
- MQTT broker to be used

Starting
--------
To start the proxy app:
node /path/to/app/slack_proxy.js

The app can also be managed by a node.js process monitor, such as [PM2](https://github.com/Unitech/pm2) or [Forever](https://www.npmjs.com/package/forever). Both have successfully been used with slack_proxy.

Usage
-----
The examples below assume the call is made from localhost, i.e. the same server where slack_proxy is running.


- **Send messages to Slack**

  Available emojis: http://www.emoji-cheat-sheet.com/

  Formatting of Slack messages: https://api.slack.com/docs/formatting

  Slack API docs: https://api.slack.com/incoming-webhooks

  - Calling using curl:

        curl -X GET 'http://localhost:8080/slack?channel=%23sense-alerts&from_user=sense-load-script&msg=Message%20To%20Send&emoji=:ghost:'

  - Calling from Sense or QlikView load script:

    First store the functions below in their own file, called for example post_to_slack.qvs.



        Sub PostToSlackInit
          // Initialize mapping table needed for URL encoding
        	URL_EncodingReferenceMap:
        	Mapping LOAD   
        		Replace([Character], 'space', ' ') as ASCII_Character,
        		//     Text([From Windows-1252]) as URL_encoding,   
        		Text([From UTF-8]) as URL_encoding
        	FROM  
        	[http://www.w3schools.com/tags/ref_urlencode.asp]  
        	(html, utf8, embedded labels, table is @1);  
        end sub


        Sub PostToSlack(vToSlackChannel, vFromUser, vMessage, vEmoji)
          SlackResult:
          LOAD
            *
            FROM [http://localhost:8080/slack?channel=%23$(vToSlackChannel)&from_user=$(vFromUser)&msg=$(vMessage)&emoji=$(Emoji)]
          (txt, codepage is 1252, embedded labels, delimiter is '\t', msq);

          drop table SlackResult;
          // trace $(vMessage);
        End Sub

    Then include the function in the load script (assuming you have a folder data connection called "Scripts", pointing to your qvs scripts). Also call the init sub to set up the mapping table used for the URL encoding:

        $(Include=[lib://Scripts/post_to_slack.qvs]);
        CALL PostToSlackInit;

    Finally, call the function from the load script. You should now get a message showing up in Slack.

        CALL PostToSlack('sense-alerts', 'server: sense-us-1', '*Financial report*: reload starting', ':information_source:');


- **Create a new directory on disk**

  Sense's default (out of the box) security model does not allow the direct interaction with the file system, as this could pose security risks. This setting can be overridden, but a better way is to keep the Sense configuration at the more secure setting, and use the slack_proxy to create the directory for you.

  - Calling using curl:

        curl -X GET 'http://localhost:8080/createDir?directory=c:/abc/def/ghi'

    *Note: The /createDir endpoint will create the entire requested directory structure. In the above example, this means that even though c:/abc did not exist, the call will create both c:/abc, c:/abc/def and c:/abc/def/ghi*

  - Calling from Sense or QlikView load script:

    Store the function below in a file called create_directory.qvs

        Sub CreateDir(vDir)
        	let vDir = Replace('$(vDir)', '\', '/');
        	let vDir = Replace('$(vDir)', '#', '%23');

        	LOAD
        		*
        	FROM [http://localhost:8080/createDir?directory=$(vDir)]
        	(txt, codepage is 1252, embedded labels, delimiter is '\t', msq);
        end sub

    Include the qvs file into the load script, the call it:

        $(Include=[lib:/Scripts/create_directory.qvs]);
        CALL CreateDir('c:/abc/def/ghi');

- **Getting disk space info**

  This endpoint will return total disk size, as well as available free space on the disk/path specified in the URL parameter.

  - Calling using curl:

        curl -X GET 'http://localhost:8080/getDiskSpace?path=/'

  - Calling from Sense or QV load script:

    Same principle as above, using a LOAD ...from [URL]. Preferably wrapping the API call in a Sub... End Sub


Warning
-------
You should make sure to configure the firewall of the server where slack_proxy is running, so it only accepts calls from the desired clients/IP addresses.
A reasonable first approach would be to configure the firewall to only allow calls from localhost.