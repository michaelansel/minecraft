Goal: make it easy to run an inexpensive, on-demand Minecraft server for friends

# Ideas
* Discord bot to launch the server (might be easier than the webapp, and also more secure than a webapp)
* Modify the cloud-init script to wait for 10m of no connections before shutting down
* Discord bot that shows the server status ([like this](https://www.reddit.com/r/discordapp/comments/8yn9hp/i_made_a_bot_that_shows_the_live_status_of_our/))
* Use an AutoScaling group to launch/terminate instances (by changing Desired Count) instead of manually creating/terminating instances
* Put everything into a CloudFormation template
* A simple webapp with a button to start the minecraft server (API Gateway, Lambda function, Poke the EC2 API, Show status on the webpage)
* Investigate an on-demand "automation-mode" server that uses a low-cost EC2 instance to just run automation; could we get away with a 512MB lightsail instance and have it be useful?
* Automatically move between automation-mode and player-mode when someone connects (need to figure out how to make reconnects work better than "you just connected, now go find the new IP")
* GitHub Actions workflow to automatically deploy changes ([something like this?](https://github.com/actions/starter-workflows/blob/master/ci/aws.yml))
* "Fork" button for a minecraft server that takes a backup and deploys a new stack running the current state
