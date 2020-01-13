# Streamflow Public Testnet

Livepeer recently deployed a new public test network to test out the upcoming Streamflow release - aimed at making Livepeer scalable, reliable, and cost effective for scaled usage. This document provides a background on the role you play in transcoding video on Livepeer, the Streamflow updates, and the steps you need to take to get up and running on the Streamflow Testnet.

Transcoding is the process of taking an input video in one format and
bitrate, and converting it into many formats and bitrates to make it
playable on the majority of devices on the planet at any connection
speed.

In the Livepeer network, users who run nodes on their infrastructure
perform this very important function, and as a result it's important
that they have high bandwidth connections, sufficient hardware, and
reliable DevOps practices. These nodes are delegated towards and
elected to perform this role, and they are rewarded with the ability
to earn fees from the network.

## Overview

This documentation will refer to two roles that infrastructure providers can play:

1. Orchestrator
2. Transcoder

An **orchestrator** is a protocol-aware, smart, 24/7 process that is responsible to the end user of the network for transcoding jobs being performed correctly. They stake LPT to secure the work that they perform, and ensure it is done correctly. They can be penalized if they maliciously cheat and mistranscode an end users content. When a user starts a Livepeer node on a CPU based server in `-orchestrator` mode, they are operating that process as an orchestrator.

A **transcoder** on the other hand, is a simple process that knows how to take an input segment of video, and transcode it to the desired outcome. It is not Livepeer protocol aware, it has no requirement for high reliability or being online 24/7, it makes no representation to the end user, and it has nothing at risk. While a user can start a Livepeer node on a server using the `-transcoder` flag to run a process in this role, they often will be running many transcoder processes, likely connected to many GPUs.

![Orchestrator and Transcoders](https://livepeer-assets.s3.amazonaws.com/bot.png)

An orchestrator distributes work across one or many transcoders.

The most popular, and default setup, is that whomever is playing the role of orchestrator, is also running many transcoder processes, and they are distributing the work only to their own processes.

While it is possible to come up with constructions for "public transcoder pools" that allow orchestrators to distribute work to random transcoders, the design space for this sort of setup is outside the scope of this documentation. This document will focus on teaching a user how to run their own orchestrator/transcoder setup to perform work on the Livepeer network. This includes: 

* How to run an orchestrator
* How to scale transcoding on an orchestrator
* How to perform transcoding on a CPU and GPU
* How to run a broadcaster
* How to migrate your setup from the Livepeer alpha to Streamflow

## Installation

First, download the latest Streamflow compatible release for your platform from the [releases](https://github.com/livepeer/go-livepeer/releases) page. If you are using OSX, download `livepeer_darwin.tar.gz`. If you are using Linux, download `livepeer_linux.tar.gz`.

After downloading the file, untar the archive and move the `livepeer` binary so that it is executable within your `$PATH`:

```
$ tar -zxvf livepeer_<YOUR_PLATFORM>.tar.gz
$ mv livepeer_<YOUR_PLATFORM>/livepeer /usr/local/bin
$ mv livepeer_<YOUR_PLATFORM>/livepeer_cli /usr/local/bin
```

## Quickstart

### Run an orchestrator

Starting `livepeer` with the `-orchestrator` and `-transcoder` flags starts the node in orchestrator mode with solo transcoding. This is the simplest and fastest way to run an orchestrator and start transcoding video on the network. The [transcoding](#transcoding) section will describe the difference between solo and split transcoding as well as how to scale transcoding on your orchestrator with split transcoding.

```
$ livepeer -network rinkeby -orchestrator -transcoder -pricePerUnit 1
```

### Run a broadcaster

Starting `livepeer` with the `-broadcaster` flag starts the node in broadcaster mode enabling you to stream video to be transcoded on the network. 

```
$ livepeer -network rinkeby -broadcaster
```

*Note that if you are already running an orchestrator node on the same machine, you will also have to pass additional flags into this command to specify unique ports so as not to conflict with your orchestrator node. See the below section on testing your transcoding setup for more detail.*

### Getting test ETH

You can get test ETH from the [Rinkeby faucet](https://faucet.rinkeby.io/).

### Getting test LPT

You can get test LPT using `livepeer_cli`. Before getting test LPT, make sure your account has test ETH.

In a separate terminal window other than the one that is running `livepeer`, run:

```
$ livepeer_cli
```

This command starts the CLI interactive wizard which can be used to issue commands to be executed by your node. The last few options of the terminal output should look something like this:

```
18. Get test LPT
19. Get test ETH
```

Select the option to get test LPT (note: the option numbering will be slightly different depending on if the wizard is connected to a node running in broadcaster or orchestrators mode). Upon entering the command in the wizard, you should see a transaction submitted by your node. After the transaction confirms, you can see your updated LPT balance by refreshing the wizard:

```
*-----------------------------*--------------------------------------------*
|                 ETH Account | 0xeb3F6d3adaA224aB84679b78376F3D96e8bF5781 |
*-----------------------------*--------------------------------------------*
|                 LPT Balance |                       10000000000000000000 |
*-----------------------------*--------------------------------------------*
|                 ETH Balance |                        9999925448000000000 |
*-----------------------------*--------------------------------------------*
```

### Open ports and networking

If you would like to test running an orchestrator or broadcaster node on the public testnet you may need certain ports open to the internet, or at least to specific IPs.

* Port **8935** (TCP) - Orchestrators need this port open to the world so that other broadcasters can discover and communicate with the orchestrator. Broadcasters should also open this port if they would like to serve their output video publicly from their node, or restrict access to this port for a specific audience or CDN.
* Port **1935** (TCP) - Broadcasters should open this port if they would like to stream into their node from a non-local source of video. You can restrict access to this port to the IP of the source of your video.


You now have a node running, and have the test ETH and LPT you need to begin interacting with the Livepeer network. Read on to learn how to activate your orchestrator node and confirm that it is transcoding video correctly.

## Transcoding

### Activation

In order to start transcoding video on the network and earning fees, your orchestrator node must be active.

An orchestrator (identified by its ETH account address) is active in a round if:

- It is registered on-chain. Registration consists of staking any amount of LPT and self-delegating. Anyone can register an orchestrator on-chain
- It is in the top 100 orchestrators with the most stake 

The active orchestrator set is locked in at the beginning of each round. Any staking activity that occurs during a round will impact membership of the active set in the following round. So, if an orchestrator accumulates stake such that it is in the top 100 orchestrators with the most stake it will join the active set in the next round. If an orchestrator previously was in the top 100 orchestrators with the most stake, but no longer is in the top 100, it will exit the active set in the next round.

You can register an orchestrator and attempt to join the active set in the next round by selecting the following option in `livepeer_cli`:

```
13. Invoke multi-step "become an orchestrator"
```

Upon selecting the option, you should be prompted to:

- Set a `rewardCut` which is the percentage of inflationary LPT rewards that you will keep (the rest will be shared with your delegators)
- Set a `feeShare` which is the percentage of ETH transcoding fees that you will share with your delegators (the rest you will keep)
- Set a `pixelsPerUnit` which is the number of pixels you define to be in a unit, which will form the basis for your charging.
- Set a `pricePerUnit` which is the price that you charge per unit of transcoding (whole number, denominated in wei). See the [configuring payment parameters](#configuring-payment-parameters) for more details on pricing per pixel.
- Set an amount of LPT to stake and self-delegate
- Set a `serviceURI` which is the public accessible IP and port that broadcasters can send requests and video to be transcoded
    - The `serviceURI` is stored on-chain in the form `https://IP:port` (when registering you will just be asked for the IP and port). The IP should remain static since orchestrators are expected to provide consistent and reliable service, but a host (DNS) name can also be used for the `serviceURI` which provides orchestrators some flexibility. Orchestrators will not be able to serve the network if they are behind a NAT (i.e. a home router). If an orchestrator is behind a NAT, you will need to make special accomodations such as enabling port forwarding or putting the orchestrator in the DMZ. Be aware that there are many risks to running a public server. You should only run an orchestrator if you are comfortable managing these risks
    - The node will check if the current public IP matches the IP stored on-chain - if there is a mismatch, then your node might not be publicly accessible. You can override the inferred public IP by starting the node with the `-serviceAddr` and using the address stored on-chain, but you should make sure that your node is actually accessible using that address 

After answering the wizard's prompt, you should see a few transactions submitted by your node. After the transactions confirm, you can see your orchestrator's registration status, stake, commission rates and pricing information by refreshing the wizard. If your orchestrator is in the top 100, it will join the active set at the beginning of the next round.

### Testing your transcoding setup 

You can test your orchestrator setup by setting up your own broadcaster and routing the broadcaster's requests directly to your orchestrator. Your orchestrator must be active before you can test the full transcoding/payment workflow - see the [activation section](#activation) for details on how to active your orchestrator. 

First, make sure to turn on verbose logging on your orchestrator (transcoding/payment related logs will not be shown with the default logging level):

```
$ livepeer -network rinkeby -orchestrator -transcoder -pricePerUnit 1 -v 99
```

Start a broadcaster that will connect directly to your orchestrator:

```
$ livepeer -network rinkeby -broadcaster -orchAddr <ORCH_SERVICE_URI>
```

`<ORCH_SERVICE_URI>` should be the publicly accessible `serviceURI` that your orchestrator registered on-chain.

Follow the steps in the [broadcasting](#broadcasting) section to deposit funds, configure broadcasting preferences and stream video into your broadcaster.

Look at the log output on your **orchestrator** node, and you should see your orchestrator start to transcode the incoming video. The broadcaster node will also receive the transcoded output back from the orchestrator and you can view your stream and each rendition in any web based or command line video player.

You can also test that your orchestrator is able to properly redeem [winning tickets](#configuring-payment-parameters) received from your broadcaster. Typically, the frequency of winning tickets in terms of clock time would be dynamically determined by the orchestrator based on the desired ticket expected value and price per pixel (these parameters are set by the orchestrator) as well as the current projected gas price required to redeem winning tickets and the amount/type of video that the orchestrator is transcoding. However, in this test scenario, since you are operating both the orchestrator and broadcaster, you can control all of the parameters mentioned to target a specific frequency of winning tickets.

First, check your orchestrator's logs to see the projected gas price for redeeming tickets (should see that the current gas price cached at a regular interval). Next, [select the renditions](#configuring-broadcasting-preferences) to request for encoding. Now, run the [PM calculator script](https://github.com/livepeer/pm-params-calculator) using the following command:

```
$ python3 calc.py -t
```

You should be prompted for the desired ticket expected value, the projected gas price for redeeming tickets, the desired time (denominated in hours) for receiving a winning ticket and the set of renditions that will be encoded. After answering all of the prompts, the script should output the price per pixel that your orchestrator should set in order to receive a winning ticket in the desired time (this is just an approximation - if the desired time is 1 hour you will not necessarily receive a winning ticket exactly in 1 hour, but rather in 1 hour on average so the time elapsed might be more or less than 1 hour in practice).

Restart your orchestrator with the values used in the script and the price per pixel outputted by the script:

```
$ livepeer -network rinkeby -orchestrator -transcoder -ticketEV <TICKET_EXPECTED_VALUE> -pricePerUnit <PRICE_PER_PIXEL> -v 99
```

Start streaming into your broadcaster and monitor the orchestrator's logs for a `redeemWinningTicket` log. Once this log is observed, you can use `livepeer_cli` and verify that the `Pending Fees` field in the wizard increased.

See the section on [configuring payment parameters](#configuring-payment-parameters) for a more detailed walkthrough of the payment parameter configuration process.

### Configuring payment parameters

The Streamflow protocol upgrade introduces two main changes to the transcoding payment flow:

- A [probabilistic micropayment](https://medium.com/livepeer-blog/streamflow-probabilistic-micropayments-f3a647672462) protocol. Broadcasters send lottery tickets to orchestrators in exchange for transcoded results. Each lottery ticket is defined with a `faceValue`, the payout to the orchestrator if the ticket wins, and a `winProb`, the probability that the ticket will win. Each ticket is treated as a micropayment worth the expected value of the ticket (calculated as `faceValue * winProb`). Orchestrators will redeem winning tickets on-chain to receive the `faceValue` of tickets.
- A payment accounting protocol that meters the resources consumed by an orchestrator during transcoding using pixels. Orchestrators define a price per pixel (denominated in wei) which is the amount an orchestrator expects to be paid per pixel transcoded. The advertised price is used by broadcasters to filter the eligible orchestrators for selection (the default broadcaster implementation currently filters out orchestrators that advertise a price that exceeds the broadcaster's own max price). The video profiles requested by a broadcaster will impact the number of pixels that need to be transcoded by the orchestrator for each video segment. The more costly in terms of pixels it is to transcode a segment, the more tickets a broadcaster will need to send to compensate the orchestrator

An orchestrator can set its price per pixel by setting its price per unit, the amount of wei to charge for each unit of work, and its pixels per unit, the number of pixels that constitute a single unit of work. An orchestrator can set its price per pixel to be fractional wei by setting the number of pixels per unit to be greater than 1. The price per pixel can be set via:

- The `-pricePerUnit` and `-pixelsPerUnit` flags when starting the node
- The `livepeer_cli` wizard by selecting the set orchestrator config option

Note: At the moment, an orchestrator will only count the number of pixels *encoded*, but not the number of pixels *decoded* during transcoding. Metering of pixels decoded will be added in a future release.

An orchestrator can set its desired ticket expected value using the `-ticketEV` flag when starting the node.

While it is running, an orchestrator will monitor the expected gas price required to confirm a transaction on-chain. Using this gas price and the desired ticket expected value the orchestrator will dynamically set the `faceValue` and `winProb` to be used for tickets to target a 1% transaction cost overhead when a winning ticket is redeemed (1% of the `faceValue` of a winning ticket covers the cost of the on-chain transaction).

The following [script](https://github.com/livepeer/pm-params-calculator) can use an orchestrator's ticket expected value, price per pixel, a gas price for redeeming winning tickets and set of video profiles to be encoded to estimate:

- The value received per hour (in terms of ticket expected value)
- The frequency of winning tickets (in terms of hours)

Find below an example of using the script to project the value received per hour and frequency of winning tickets when transcoding 10 streams into 240p, 360p and 720p renditions:

```
➜  pm-params-calculator git:(master) python3 calc.py -f
DEFAULTS
---------
Ticket redemption gas cost: 100000
Transaction cost overhead: 0.01


Enter the desired ticket expected value (gwei): 1000


Enter the gas price (gwei) to use for ticket redemption transactions: 5
Ticket redemption gas price: 5.0 gwei


Transaction cost to redeem a ticket: 0.0005000000 ether ($0.1000000000)
Ticket face value: 0.0500000000 ether ($10.0000000000)
Ticket winning probability: 0.000020000000000
1 out of 49999 tickets will win


Enter the price per pixel (wei) to charge: 1000
Price per pixel: 1000 wei


Would you like to add a rendition to be encoded? (y/n): y
Enter the output width: 426
Enter the output height: 240
Enter the output FPS (frames per second): 30
Enter the number of streams of this renditions: 10
Would you like to add a rendition to be encoded? (y/n): y
Enter the output width: 640
Enter the output height: 360
Enter the output FPS (frames per second): 30
Enter the number of streams of this renditions: 10
Would you like to add a rendition to be encoded? (y/n): y
Enter the output width: 1280
Enter the output height: 720
Enter the output FPS (frames per second): 30
Enter the number of streams of this renditions: 10
Would you like to add a rendition to be encoded? (y/n): n


Given the specified renditions you will:
Encode 1354579200000.0 pixels per hour
Receive 1354 tickets per hour
Receive 0.0013540000 ether ($0.2708000000) (in terms of ticket expected value) per hour
Receive 1 winning ticket every 36.92688330871492 hours
```

In the above example, if an orchestrator transcodes 10 streams into 240p, 360p and 720p renditions, sets the ticket EV to 1000 gwei, redeems winning tickets on-chain using a gas price of 5 gwei and sets the price per pixel to 1000 wei then the orchestrator will receive approximately .001354 ETH of value per hour and 1 winning ticket every ~36.929 hours.

The script can also be used to calculate the price per pixel needed to target a desired frequency of winning tickets (in terms of hours) which can be useful for testing that your orchestrator can properly redeem winning tickets.

```
➜  pm-params-calculator git:(master) python3 calc.py -t
DEFAULTS
---------
Ticket redemption gas cost: 100000
Transaction cost overhead: 0.01


Enter the desired ticket expected value (gwei): 1000
Ticket expected value: 1000.0 gwei


Enter the gas price (gwei) to use for ticket redemption transactions: 5
Ticket redemption gas price: 5.0 gwei


Transaction cost to redeem a ticket: 0.0005000000 ether ($0.1000000000)
Ticket face value: 0.0500000000 ether ($10.0000000000)
Ticket winning probability: 0.000020000000000
1 out of 49999 tickets will win


Enter the desired number of hours (i.e. 1, .5, etc.) until receiving a winning ticket: 1


Would you like to add a rendition to be encoded? (y/n): y
Enter the output width: 426
Enter the output height: 240
Enter the output FPS (frames per second): 30
Enter the number of streams of this renditions: 10
Would you like to add a rendition to be encoded? (y/n): y
Enter the output width: 640
Enter the output height: 360
Enter the output FPS (frames per second): 30
Enter the number of streams of this renditions: 10
Would you like to add a rendition to be encoded? (y/n): y
Enter the output width: 1280
Enter the output height: 720
Enter the output FPS (frames per second): 30
Enter the number of streams of this renditions: 10
Would you like to add a rendition to be encoded? (y/n): n


Given the specified renditions you will and a target of 1 winning ticket every 1.0 hours you will:
Need to charge 36911.09386590315 wei per pixel
Encode 1354579200000.0 pixels per hour
Receive 49999.0 tickets per hour
Receive 0.0499990000 ether ($9.9998000000) (in terms of ticket expected value) per hour
```

In the above example, if an orchestrator transcodes 10 streams into 240p, 360p and 720p renditions, sets the ticket EV to 1000 gwei, redeems winning tickets on-chain using a gas price of 5 gwei and targets 1 winning ticket every hour then the orchestrator should set the price per pixel to 36911.09386590315 wei.

In practice, the price set by orchestrators will likely also be influenced by the market rate for transcoding on the network and an orchestrator's own infrastructure cost. This information can be surfaced by tools that will be built in the future.

### Scaling transcoding

The easiest way to setup an orchestrator to transcode video is to have the orchestrator perform solo transcoding by passing in both the `-orchestrator` and `-transcoder` flags when running `livepeer`. However, if you want to scale out your transcoding operation to support many concurrent streams, you will want to enable split transcoding.

Split transcoding consists of running many individual transcoder nodes (usually on separate remote machines) that connect to your orchestrator. When your orchestrator receives video from a broadcaster, it will then distribute the incoming segments to the invidual transcoders instead of transcoding the video on its own. An orchestrator can increase its capacity and scale horizontally by adding more transcoders to its backend.

In order to enable split transcoding, you should first run an orchestrator with solo transcoding turned off:

```
$ livepeer -network rinkeby -orchestrator -pricePerUnit 1 -orchSecret <ORCH_SECRET>
```

`<ORCH_SECRET>` is a secret defined by the orchestrator that is used to authenticate requests from transcoders. The secret should be shared with all transcoders to be attached to the orchestrator.

Next, you should attach a transcoder to your orchestrator:

```
$ livepeer -transcoder -orchAddr <ORCH_SERVICE_URI> -orchSecret <ORCH_SECRET>
```

`<ORCH_SECRET>` should be the secret defined by your orchestrator. `<ORCH_SERVICE_URI>` should be the publicly accessible `serviceURI` that your orchestrator registered on-chain.

You can run the above command on any number of machines that you would like to dedicate to transcoding.

### GPU transcoding

When you setup split transcoding for your orchestrator, you can run individual transcoders on not only CPUs, but GPUs as well. The GPUs that transcoders run on can be dedicated purely to transcoding or can also be mining cryptocurrencies simultaneously. See [this guide](https://github.com/mk-livepeer/bot-miner-setup) for instructions on how to setup transcoders on GPUs.

## Broadcasting

### Deposit broadcasting funds

You will need to deposit funds (ETH) used to pay orchestrators on the network for transcoding video.

In `livepeer_cli`, select the following option:

```
1.  Invoke "deposit broadcasting funds" (ETH)
```

Upon selecting the option, you should be prompted to entire the amount of ETH to allocate for your deposit and reserve. Broadcasting funds are split into a deposit and a reserve. Deposit funds are used to pay any active orchestrator on the network. Reserve funds guarantee active orchestrators up to a fixed cap to ensure that orchestrators are paid fairly even if a broadcaster depletes its primary deposit. The distinction between the deposit and the reserve arises from the probabilistic micropayment protocol that broadcasters use to pay orchestrators - see [this blog post](https://medium.com/livepeer-blog/streamflow-probabilistic-micropayments-f3a647672462) for more details. 

After answering the wizard's prompt, you should see a transaction submitted by your node. After the transaction confirms, you can see your updated deposit and reserve by refreshing the wizard.

### Withdraw broadcasing funds

When you want to withdraw your broadcasting funds you will need to wait a fixed number of rounds before gaining access to your funds. The first step for withdrawal is to request to unlock your funds.

In `livepeer_cli`, select the following option:

```
13. Invoke "unlock broadcasting funds"
```

Upon selecting the option, you should be prompted to confirm that you would like to request to unlock your funds. After answering the wizard's prompt, you should see a transaction submitted by your node. After the transaction confirms, you will be able to withdraw your funds at the withdraw round that was specified in the prompt.

If you change your mind and do no want to withdraw your funds, you can cancel your unlock request in `livepeer_cli` by selecting the following option:

```
14. Invoke "cancel unlock of broadcasting funds"
```

Upon selecting the option, you should be prompted to confirm that you would like to cancel your request to unlock your funds. After answering the wizard's prompt, you should see a transaction submitted by your node. After the transaction confirms, your unlock request will be cancelled. Another way to cancel an unlock request is to deposit more funds.

If you want to withdraw your funds, in `livepeer_cli` select the following option:

```
15. Invoke "withdraw broadcasting funds"
```

If your unlock is not complete (i.e. your withdraw round is in the future), you will not be able to withdraw. Once your unlock is complete (i.e. your withdraw round is the current round or in the past), you should be prompted to confirm that you would like to withdraw your funds. After answering the wizard's prompt, you should see a transaction submitted by your node. After the transaction confirms, your withdrawal will be complete and can see your empty deposit and reserve by refreshing the wizard.

### Configuring broadcasting preferences

You can configure the following broadcasting preferences using the wizard:

- The maximum price per pixel you are willing to pay. See the orchestrator section on [configuring payment parameters](#configuring-payment-parameters) for more details on how pricing per pixel works
- The set of video profiles that you want your input video to be transcoded into

In `livepeer_cli`, select the following option:

```
16. Set broadcast config
```

First, you will set the maximum transcoding price. You will be prompted for a maximum price relative to certain number of pixels. For example, you can set your maximum price to 0.1 wei per pixel if you set the maximum price to 1 wei for 10 pixels. The default number of pixels will be 1.

Then, you will pick a set of video profiles that you would like your input video to be transcoded to. The set of video profiles presented in the wizard correspond to the [standard video profiles](https://github.com/livepeer/lpms/blob/master/ffmpeg/videoprofile.go) currently supported by the node. Future releases will offer greater flexibility around customizing video profiles to use for transcoding.

### Broadcasting video

See the [broadcasting guide](https://livepeer.readthedocs.io/en/latest/broadcasting.html) for information on sending video into your broadcaster node and viewing the output transcoded video. In order to see more detailed logs, run your broadcaster with `-v 99` to enable verbose logging.

After receiving a stream, your broadcaster will try to connect to a set of orchestrators (ether based on the registered orchestrators on-chain or based on the orchestrators specified using the `-orchAddr` flag). Your broadcaster might reject certain orchestrators based on their required price or ticket parameters. 

If you observe the `ticket faceValue higher than max faceValue` error in the broadcaster's logs, you can try:

1. Running the broadcaster with `-depositMultiplier 1`. The default value is 1000 (which is pretty high) meaning that the broadcaster will not be willing to use a ticket `faceValue` that exceeds the broadcaster's deposit divided by 1000. So, if the value is 1 then the broadcaster will be willing to use a ticket `faceValue` that equals the broadcaster's deposit. While this might not be desirable in other circumstances, it can be fine for testing purposes
2. If option 1 does not work, then the broadcaster's deposit is less than the `faceValue` required by an orchestrator - you should try depositing more ETH

If you observe the `ticket EV higher than max EV` error in the broadcaster's logs, you can try running the broadcaster with `-maxTicketEV <MAX_TICKET_EV>` where `MAX_TICKET_EV` is the maximum expected value (denominated in wei) for tickets sent by the broadcaster. The default value is 10 gwei so you could try using a higher value.

### Transcoding verification (experimental)

The broadcaster can verify transcoded results received from orchestrators by sending verification requests to a verifier that runs alongside the broadcaster. The current default verifier implementation uses a machine learning classifier - refer to the [verifier documentation](https://github.com/livepeer/verification-classifier) for more details. 

Transcoding verification is currently experimental and needs to be explicitly enabled by the user. R&D to improve its accuracy (in classifying correctly vs. incorrectly transcoded video) and performance (in terms of run time and computation cost) is underway, but in the short term, expect to see higher computational costs and transcoding latency after enabling verification for a broadcaster.

In order to enable transcoding verification, first make sure that you have [Docker](https://docs.docker.com/install/) installed and setup the verifier.

```
git clone https://github.com/livepeer/verification-classifier
cd verification-classifier
./launch_api.sh
```

`./launch_api.sh` will start a verifier server running within a Docker container that accepts verification requests at the `/verify` endpoint. You should see log output that indicates that the verifier is listening on port 5000:

```
Sending build context to Docker daemon  5.904MB
Step 1/4 : FROM verifier:v1
 ---> fe5d1924dab5
Step 2/4 : COPY /api ./scripts
 ---> Using cache
 ---> bc21bfb203ef
Step 3/4 : ENTRYPOINT ["python"]
 ---> Using cache
 ---> 816b804bc7e7
Step 4/4 : CMD ["scripts/api.py"]
 ---> Using cache
 ---> 548f58689798
Successfully built 548f58689798
Successfully tagged verifier-api:v1
Verifier server listening in port 5000
```

If you are running the verifier on the same machine as the broadcaster, then you can run the below command in order to connect the broadcaster with the verifier:

```
livepeer -network rinkeby -broadcaster -verifierUrl http://localhost:5000/verify -verifierPath ~/verification-classifier/stream
```

The broadcaster logs should indicate the address of the verifier being used:

```
Using the Epic Labs classifier for verification at http://localhost:5000/verify
```

The value of `-verifierUrl` should be the address (`http://<IP>:<port>/verify`) that the verifier is receiving requests at. In this case, since the verifier and broadcaster are on the same machine all requests can be received on `localhost`.

The value of `-verifierPath` should be the path to the volume that the verifier will read segment data from. By default, the verifier will read segment data from a volume mounted on the `/stream` directory within the `verification-classifier` project. So, if you started the verifier from within `~/verification-classifier` the value for `-verifierPath` should be `~/verification-classifier/stream`.

If you are running the verifier on a separate machine from the broadcaster, you will need to make sure that the verifier endpoint is accessible by the broadcaster and that the verifier's volume is accessible over the network.

One way to make the verifier endpoint accessible to the broadcaster without making it publicly accessible is to create an SSH tunnel from the broadcaster machine to the verifier machine.

Create an SSH tunnel with the following command run from the broadcaster machine:

```
ssh -N -L 5000:127.0.0.1:5000 -i <SSH_KEY_FILE> <USER>@<HOSTNAME>
```

Note that the command will not output anything if it is successfuly run.

One way to make the verifier's volume accessible over the network is to mount the verifier's volume on the broadcaster machine over an SSH connection using [sshfs](https://github.com/libfuse/sshfs).

Mount the verifier's volume on the broadcaster machine by running the following command on the broadcaster machine:

```
sshfs -o IdentityFile=<SSH_KEY_FILE> <USER>@<HOSTNAME>:<FOLDER_ON_VERIFIER> <FOLDER_ON_BROADCASTER>
```

After the SSH tunnel is setup and the remote volume is mounted, run the following command to start the broadcaster:

```
livepeer -network rinkeby -broadcaster -verifierUrl http://localhost:5000/verify -verifierPath <FOLDER_ON_BROADCASTER>
```

The broadcaster also supports sharing segment data with a verifier using external cloud storage providers such as Amazon's S3 or Google's GCS. Documentation on using external cloud storage providers will be added in the future.

## Running an Ethereum node

By default `livepeer` will use Infura as the ETH RPC provider. If you would like to run your own ETH node you can use `geth` (the Rinkeby testnet is only supported by `geth`) which can be installed using the instructions on the [installing geth page](https://geth.ethereum.org/docs/install-and-build/installing-geth). 

Run `geth`:

```
$ geth -rinkeby -rpc -rpcapi eth,net,web3
```

If `geth` is running on a different machine than `livepeer` you will have to specify the `-rpcaddr` flag to indicate the interface to listen on.

Wait until `geth` is fully synced with the latest block on the Rinkeby testnet. You can check if `geth` is done syncing by using the Geth Javascript Console:

```
$ geth attach http://localhost:8545
Welcome to the Geth JavaScript console!

instance: Geth/v1.9.0-stable-52f24617/linux-amd64/go1.12.7
coinbase: 0x0161e041aad467a890839d5b08b138c1e6373072
at block: 583 (Wed, 23 Oct 2019 17:41:00 EDT)
 modules: debug:1.0 eth:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> eth.syncing
false
```

In order to connect to your own ETH node, you will need to set the `-ethUrl` flag on your `livepeer` node to the ETH node's RPC URL.

For example, you could use the following command to connect an orchestrator to an ETH node running at `localhost:8545`.

```
$ livepeer -network rinkeby -orchestrator -pricePerUnit 1 -ethUrl http://localhost:8545
```

## Migrating to Streamflow

There are a number of backwards incompatible changes in a Streamflow node relative to a Livepeer alpha node. While you can start fresh on the Streamflow public testnet with a completely new ETH account address, it is recommended that you take certain steps to ensure that running a Streamflow node does not inadvertently modify or in the worst case delete sensitive files (i.e. keystore files) used by a Livepeer alpha node:

- Backup all files used by your Livepeer alpha node. These files are in the node's data directory. The default data directory for a Livepeer alpha node is `~/.lpData`. If you specified a custom data directory via the `-datadir` flag, then the files will be in the directory specified
- If you want to preserve the data directory used by your Livepeer alpha node, you should run your Streamflow node with a custom data directory specified via the `-datadir` flag
- It is important that you start your Streamflow node with a fresh database file. The default name for the database file is `lpdb.sqlite3`. The easiest way to ensure that your Streamflow node starts with a fresh database file is to start the node with an empty data directory (the node will automaticaly create a fresh database file in this case)
- Since the Streamflow public testnet involves a completely new set of contracts, it is fine to use newly generated ETH accounts

## FAQ

Quicklinks:

[Forum](http://forum.livepeer.org/)

[Chat](https://discord.gg/cBfD23u)

[Streamflow paper](https://github.com/livepeer/wiki/blob/master/STREAMFLOW.md)