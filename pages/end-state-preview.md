---
layout: center
---

# What are we _actually_ doing though?
And how about some alternatives?

---

## A purple-y project
- ASP.NET core API
- API talks to a React front-end
- API talks to a SQL server database
- Integration tests running against a 'real' database

---

## Pipelines in source, running CI and CD

![pipelines](/images/pipelines.png)

---

## Why not have 1 pipeline?
We could have the whole CI/CD in ONE pipeline, but there are trade-offs

- The pipelines appears 'unfinished' unless it's been deployed to the highest environment available
- A change to the 'Test' deployment would require running the build and deploying to dev just to verify the change

--- 

## So why not just use the releases view?
- More transparent deployments, but less transparent changes
- It's not in your project repo
- Harder to make changes affecting multiple environments
- Variables are hidden away 🤷
- Microsoft are pushing towards yaml pipelines

---

## Octopus deploy?
- I'm no expert here
- \$\$\$
- 🤷
<!-- TODO: Learn more about octopus deploy -->

---

## Wait, once more, what are we doing?
I'm going to talk about this way of doing things

![pipelines](/images/pipelines.png)