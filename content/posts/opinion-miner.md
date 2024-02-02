---
title: "Micro-SaaS on the horizon?"
date: 2024-02-01T14:30:58+01:00
tags: [ "python", "llm", "rust" ]
---

Code for all the examples can be found on [this repository](https://github.com/NikolaMilosa/opinion-miner/tree/10bbff3dda728469842d6a681d35150f9d0cf404).

This has to be the first time my head flooded with ideas on how can this be turnt into a profitable project. Even a micro-saas! Maybe this blogging thing is starting to produce its benefits! This weeks work was directed towards a PoC for an opinion miner that I wanted to create. The idea is the following:
> When someone needs to find out what does the general public think of a certain topic who to ask?

Google-ing is tedious as you need to spend hours reading blogs and the problem is that even if you do that a lot of content is AI generated and you really waste your time there. So how about automating it?

## The problem

I wanted to build a system that can be easily used and the whole point of it is to get all sorts of data and sentiment analysis about a certain topic. I see this extreamly fit for small to mid businesses that require data about the market and want to get a feel for a certain topic. The underlying topic of sentiment analysis is something that can be tackled in many ways and probably should be. Since the automation is used for something that is really *subjective* a vast amount of data should be provided that were gathered with different techniques so the user can decide wether to trust the data or not.

So to define everything for this PoC I wanted to be able to fetch data, analyze it and draw a graph of it so the user has the idea what is the general opinion based on the public. Lets break it into some steps.

### Data fetching

I knew from the start that this has to be as modular as possible since I am going to expand on it in some future and this part was done in rust. It is a cli tool that can be used to fetch data from different sources. Right now it fetches blogs from [dev.to](https://dev.to/). They've recently openned their [forem](https://developers.forem.com/api) for articles and allowed us to use it which is pretty nice! I really like the site and I read it from time to time. To be quite honest I use [daily.dev](https://daily.dev/) as the source of my developer blogs but both of those are quite good at what they do and dev.to seemed to have an open API so I would like to thank them for that!

I've chosen rust because it is quite nice for building small cli tools and it can be statically compiled with a super small sized binary. For the future my idea is to have a main API which orchestrates between these small programs and runs them on demand when a user asks about a certain topic. The binary as of now consists of a simple [tokio](https://tokio.rs/) main which parses the arguments of the cli, creates a `cancellationToken` (which I think are way nicer than channels for graceful shutdowns) and spawns a thread for a `DevToFetcher`.
```rust
info!(logger, "Running fetchers...");
let cancel_token = CancellationToken::new();

let dev_to_fetcher = DevToFetcher::from_cli(&cli, cancel_token.clone())?;
let dev_to_fetcher_logger = logger.clone();
let dev_to_fetcher_handle =
    tokio::spawn(async move { dev_to_fetcher.run(dev_to_fetcher_logger).await });

tokio::select! {
    _ = tokio::signal::ctrl_c() => {
        info!(logger, "Received shutdown request...");
        cancel_token.cancel()
    }
}

dev_to_fetcher_handle.await.unwrap();

info!(logger, "Finished...");
Ok(())
```
As the code grows and when we add more fetchers I believe that modifications for creating handlers and their termination will change to a `Vec<T>` whete `T` will be a boxed trait `Fetcher`. The whole trait right now looks poor and I am not that happy with how it turned out and probably will be changed.
```rust
pub trait Fetcher: Sized {
    async fn run(&self, log: Logger);

    fn from_cli(cli: &Cli, token: CancellationToken) -> Result<Self, Error>;
}
```
I'm in the motion of writing clean simple cli's that are really good at doing one thing and I chose the output of this cli to be a standard out. That way you can easily pipe it to another program, or even spawn it from something else and captcure its output as it is eventually comming. 

As for the code of the fetcher itself it will implement all sorts of platforms and ideas. One fetcher that I've implemented is a simple API fetcher and it uses `reqwest` to fetch the data. The only problem I've had that I think forem communicated poorly is that you have to specify a `User-Agent` header or it will return access denied even if you don't really need the token. To create the fetcher from cli implementation is as follows:
```rust
fn from_cli(cli: &crate::cli::Cli, token: CancellationToken) -> Result<Self, Error> {
    let mut headers = HeaderMap::new();
    headers.append(
        "accept",
        HeaderValue::from_str("application/vnd.forem.api-v1+json")
            .expect("can create header value"),
    );
    headers.append(
        "user-agent",
        HeaderValue::from_str("rust-code").expect("can create header value"),
    );
    Ok(Self {
        client: reqwest::Client::builder()
            .default_headers(headers)
            .build()
            .map_err(|e| Error::Invalid(e.to_string()))?,
        base_url: cli.dev_to_url.clone(),
        per_page: cli.dev_to_page,
        token,
        term: cli.term.to_string(),
    })
}
```

Here nothing fancy is stated but I do like that I can reuse the client and really have everything preconfigured in terms of headers and authentaction. The fetching right now works in a simple loop. With this code I started to realize the things I don't like about `tokio::select`. If I want to do correct and immediate shutdown I have to do checking of cancelation token all the time and on every future. One could say that I could've finished the whole batch and then exit but I like the quickest exit possible.
```rust
loop {
    /// Prepare request, I like to do it like this just so I can print the debug of it if needed
    /// and usually... Its needed
    let request = self
        .client
        .get(url.clone())
        .query(&[
            ("page", &num_scrape.to_string()),
            ("per_page", &self.per_page.to_string()),
        ])
        .build()
        .expect("can build a request");

    debug!(log, "Full request: {:?}", request);

    /// Tokio select numero uno
    let response = tokio::select! {
        _ = self.token.cancelled() => {
            debug!(log, "Received shutdown");
            break;
        }
        response = self
        .client
        .execute(request) => {
            match response {
                Ok(r) => r,
                Err(e) => {
                    debug!(log, "Failed scrape with error: {:?}", e);
                    continue;
                }
            }
        }
    };

    num_scrape += 1;

    debug!(log, "Successfully scraped dev.to");

    /// Tokio select numero dos
    let parsed = tokio::select! {
        _ = self.token.cancelled() => {
            debug!(log, "Received shutdown");
            break;
        },
        parsed = response.json::<Vec<ArticlePreFetch>>() => {
            match parsed {
                Ok(v) => v,
                Err(e) => {
                    debug!(log, "Failed to decode with error: {:?}", e);
                    continue;
                }
            }
        }
    };

    for article in parsed {
        /// In fetch_content call we have a couple more tokio selects
        /// At this point I've realized that maybe I should've just ended on a whole batch
        let content = match self.fetch_content(article.id, &log).await {
            Ok(c) => c,
            Err(Error::Shutdown) => break,
            Err(e) => {
                debug!(log, "Error while fetching article content: {:?}", e);
                continue;
            }
        };
        if !content.body_html.contains(&self.term) {
            continue;
        }
        let full_article = Article {
            body_html: content.body_html,
            description: article.description,
            id: article.id,
            title: article.title,
            source: "dev.to".to_string(),
        };
        println!(
            "{}",
            serde_json::to_string(&full_article).expect("can serialize into json")
        )
    }
}
```
### Sentiment analysis

The real juicy part starts somewhere here. At least it did for me. There is multiple ways how I could approach sentiment analysis:
1. Use `nltk`. This is something that I want to try and add to the mix. 
2. I could train my own AI which could do something like that.
3. Or I could just ask somebody else to do it? And do it for free (-ish)...

This is the first time where I wanted to deploy a local instance of some LLM and try to integrate it in the solution. So technically I went with the third option. A quick Google search later I found [modelz](https://github.com/tensorchord/modelz-llm) which I could easily port into my python script and *voila* I have myself a local chatbot! On their repository they state the models they support and what are the recommended settings for running. Since my work laptop doesn't have a GPU I was left to play with two CPU models that come from bloomz. They don't have a lot of parameters but for a demo they are nice. What I like the most about it is that it was easy to test. For a production service you probably wouldn't use your own chatbot since you need some advanced hardware stuff and you are better off paying some money than trying to do it yourself.

Since the text that I scraped was html I used [beautifulsoup](https://pypi.org/project/beautifulsoup4/) to extract just the relevant text. After that I had to do some tweaking for it to work on a local machine. I didn't figure out why but when I ask a local LLM for a completion based on a larger text it just crashes so what I did instead is buffer them in batches of 3.
```python
total= []
for i in range(0, len(sentences), 3):
    if not any([search_term in sentence for sentence in sentences[i:i+2]]):
        continue
    messages = [{
        "role": "user",
        "content": f"""You are a research scientist and your role is to devise the sentiment of a search term in a given text. The search term will be a couple of words and the text will be scraped text. The sentiment values that you have to ONLY return can be one of 'Really Positive', 'Positive', 'Neutral' 'Negative', 'Really Negative'. You will be sent a lot of sentences and when you receive a message 'Decide' you should return one work from return options.
        For example: 
        user:'I use Python somewhat regularly, and overall I consider it to be a very good language. Nonetheless, no language is perfect.'
        user: 'Decide'
        You should return 'Positive'

        Given the previous instruction devise the sentiment value of '{search_term}' in the following sentences:
        """
        }] + [{
            "role": "user",
            "content": sentence
        } for sentence in sentences[i:i+2]] + [{
            "role": "user",
            "content": "Decide"
        }]
    completion = client.chat.completions.create(
        model= "any",
        messages= messages)
    guess = completion.choices[0].message.content.strip().lower()
    decided = 'neutral'
    for sentiment in sentiments:
        if sentiment in guess:
            decided = sentiment
            break

    total.append(decided)
```
To explain the above code lets say that the `search_term` is equal to *testing* and `sentences` is an array that consists of some sentences that have the `search_term` in them. If the whole batch (in our case 3) sentences don't have the `search_term` we don't need the LLM's opinion on it. If the batch contains that term we provide instructions to the chatbot and tell him what to do. This is the part which I should tweak and see if I can get better results. The larger the model the more it understands what I want it to do. Notice that at the end of the code we have a checker that checks if LLM returned anything from the range of sentiments I specified. Sometimes it returned only the sentiment, sometimes it returned a sentence with the decision and sometimes it returned something random. I think that saying if LLM returned nothing plausible is `neutral` is a bit far-fetched 😆.

The only left part is to decide from the `total` what is the resulting sentiment of the text and I did that in a most briliant way possible. Basically i returned the most occuring result. 
```python
...
return max(set(total), key=total.count)
```
I think this is not the best way to do that and I should somehow think of a way to add weights to sentiments. I've had a lot of neutral results that were decided by one from the array and it was probably that the LLM returned some trash and it rolled it as neutral...

### Data presentation
As for this part and since this is a PoC I decided to pick something easy and I went with simple plotting of the numbers. I've had some strugles with rerendering of matplotlib but I managed to fight those. Didn't know it can be so unintuitive to do such a small thing. This is the part that will surely become better as the solution matures and I will look to provide as much data as possible. This data can be plotted in a lot of ways. The PoC that is implemented on this commit is a simple bar chart but there will be a lot more. 

## How does this tie together
The story of tying all this together is a fun one. I find it funny how all PoCs I did usually end up super weird. I've gone with a simple python script that calls all the things in separate subprocesses and threads.
```python
stop_event = Event()
logger.info("Spawning api thread with model '%s'...", args.model)
api_thread = Thread(target=run_api, args=(stop_event, args.model, args.use_cpu))
api_thread.start()

# Wait for the API to start
while True:
    logger.info("Waiting for api to start...")
    time.sleep(1)
    try:
        response = requests.get("http://localhost:8000")
        if response.status_code == 200:
            logger.info("API started!")
            break
    except Exception:
        continue
    
logger.info("Configuring plot...")    
```
Once the API is up and running we can call the scraping thread and give it all the parameters it needs to start scraping.
```python
queue = Queue()

logger.info("Spawning thread to scrape blogs for search term '%s'...", args.term)
scrape_thread = Thread(target=run_cli, args=(stop_event, queue, args.term, args.sample_size, logger))
scrape_thread.start()

while True:
    try:
        line = queue.get()
        process_line(line, memory, args.term, bar, figure, axes, logger)
    except KeyboardInterrupt:
        logger.info("Received interupt...")
        break
    
logger.info("Stopping threads...")
stop_event.set()

api_thread.join()
scrape_thread.join()
logger.info("Threads stopped")
```
Since there aren't really `Channel`s in python I've went with a simple `Queue` between the main loop and the cli loop where the cli loop reads data and puts it in the queue, and main thread is reading the said data and display it in a plot. This part will probably be separated into multiple microservices for the production setup.

## Conclusion
I finally feel like I have a meaningful idea. This project will be fun as it grows and I will for sure write more posts about it and will definitely dive into this. I think all developers should have some kind of drive after work. Work is a place of stress and tention whereas our side-projects are our passion and free time. I have a strong desire to create and its an uncontrollable need for me.

I've been asked a couple of times by non-tech friends who feel like we do magic something like:
> Aren't you afraid that someone will steel your ideas and make huge profits off of it?

Honestly I don't know. I still haven't created anything profitable and don't know how that will look like. I think that it will be a huge compliment if someone copies my idea and sells it better. I am not a salesperson I am a developer and that is something that one has to learn.

Until the big jump in the world of micro-saas, cheers!
