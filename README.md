# Vega specs
This repository contains specifications for Vega protocol. 
At this point in time not all specifications are fully implemented and there are parts of Vega that do not have a counterpart specification.
It is very much work in progress. 

If you think any of the specifications are unclear or incorrect please raise an issue in this repo. 
If you would like to see an improvement / feature of the protocol please raise an issue in this repo to open a discussion on the topic. 

We are currently not accepting pull requests on this repo.

## [Protocol](./protocol/)
This directory contains the protocol specifications. The Platonic ideal is that this directory contains all that is needed to write an
implementation of Vega that is compatible with [vegaprotocol/vega](https://github.com/vegaprotocol/vega) (currently private repo).

## [Glossaries](./glossaries/)
These are quick reference points for general terminology we use. Some of the specs are really dense with trading terms 
or blockchain specifics. If something comes up that you don't understand and have to look up, it's likely it will happen
to someone else - raise an issue. 
The advantage of adding it here rather than
letting people go off and search on their own is that we can point out if a specific feature/term is applied differently
due to Vega's design.

## [QA-scenarios](./qa-scenarios/) 
Scenario files that cover some of the acceptance criteria of various spec files. 
The ones with `.feature` extension are runnable with cucumber/godog test harness that is part of [vegaprotocol/vega](https://github.com/vegaprotocol/vega) (currently private repo).
