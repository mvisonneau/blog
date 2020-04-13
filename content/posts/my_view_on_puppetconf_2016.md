+++
title = "My view on PuppetConf 2016"
description = "Some thoughts around the talks I was able to attend at PuppetConf 2016 - San Diego, CA"
tags = [
  "puppet",
  "puppetconf",
]
date = "2016-11-19"
categories = [
  "puppet",
  "conferences",
]
highlight = "true"
+++

## TL:DR

I was lucky enough to be able to spend 4 days of October in San Diego attending PuppetConf 2016. Don’t worry though ! As appealing as it seems, even if it was sunny and warm we were pretty much settled in dark rooms with the AC running at 200% talking about configuration management all day long. Here is a summary of what I believe are the most valuable things we were able to learn during the event.

## Contributor Summit

It all started with the Contributor Summit. This is an all-day event that gathers people that are already contributing to Puppet or that are willing to. This day is the opportunity to discuss about the current state of the solution and its components with the people we are use to work with during the year over the internet. This year, there was about 300 people, with a lot of them from Puppet and Voxpupuli. If you don’t know Voxpupuli, it is a group of contributors that manages loads of interesting puppet modules on the forge.

During this day I mostly participated in the discussions regarding configuration management for containers and testing improvements related topics. There was several other topics such as network devices, windows, development best-practices or some module coding hackathon.

Surprised to discover that most of the people present that day were still running Puppet 3.x or even 2.7. It seems that they are really tied with their current implementations. Interesting to see how people can lock themselves with products so quickly.

## Grand opening

The official beginning of the conference started on the next day with about all conference attendants gathered in a huge room. Luke Kanies, founder and former CEO kicked it off at 9am sharp. He recently (September 2016) left his position to Sanjay Mirchandani who previously worked on C levels at Microsoft, EMC and VMWare.

Luke and Sanjay, one after the other managed to convince that Puppet was not on the decline yet and reviewed the basics advantages of using their product. They seemed both scared by the rise of containers into tech companies but showed some usages of Puppet applied to it. First one, managing scheduling of containers which wasn’t very convincing. Second one, managing the build process of containers which made much more sense.

There also was a lot of marketing with the demo of Puppet Enterprise dedicated features by some of their engineers. As well as some feedbacks from Nike CIO, one of their biggest paying customer .

## Marc Cluet’s talk around Service Discovery

Marc was one of the first speakers, starting right after the main opening. A lot of people were present to attend his presentation around how to manage and use service discovery with Puppet. He reviewed the general definition of service discovery and the current solutions available on the market. He then followed up focusing more specifically on how Consul works and how he managed to get it to work with Puppet.

## Talks on Kubernetes, Docker, Best Practices in the cloud

### Best Practices for Puppet in the Cloud by Randall

The idea of not using shell scripts within containers is something that is very interesting and I am willing to dig deeper into it. However I diverge on the fact that containers images as immutable as they are should be able to be configured at runtime

### Gary Larizza’s talk around roles/profiles best practices

Gary’s talk worth the trip itself as he is always doing the show! He talked about best practices around puppet code management using the role/profiles model, feedbacks he had from people who implemented it and how it changed his point of view regarding it. Very constructive, the conclusion is that there is not a single and strict way of implementing this model. It has to be from the opinion of the people managing puppet codebase to decide how they want to do it in order to let them work as efficiently as possible.

### Seth Vargo’s talk around secret management and Vault

Seth is a technical director at Hashicorp. His talk was covering the current state of security management in most of companies and how Vault can help you being not only more secure but also more agile in the way you manage your secrets. It was an amazing presentation, very interactive that was completely packed. I wouldn’t risk myself paraphrasing him so my only recommendation is to checkout this awesome project : [https://www.vaultproject.io/](https://www.vaultproject.io/)

### My talk around scaling puppet on AWS using Terraform and ECS

I was quite stressed out at the beginning given the crowd but it turned out well. I think I managed to proudly represent the colours of Trainline and I hope that it was of some interest for the people attending. I’ve had some very warm feedbacks after my talk regarding the job that I had done. I might do a burrito session in the upcoming weeks in which I will be redoing this talk. Otherwise, it was recorded and I believe that it should be available by November 7th.

To conclude, it seems like there are valid initiatives from Puppet(labs) to keep their product attractive in the future. Let’s wait and see how the tendance goes with the container implementation in the cloud world. I think that even it hasn’t been mentioned yet, they will have to review their pricing policies in order to attract more paying customers and maintain their existence.

Thanks for having reached out the end of it, hope it was of some interest for you!
