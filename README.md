# Clout Improvement Proposals (CIPs)

## Context

[BitClout](bitclout.com) is a decentralized social network. All of the code
and data are totally open and anyone can build on the network or
contribute code to improve it.

The CIPs repo allows developers to submit proposals for major features to get feedback and
buy-in from the community prior to submitting code for review.

 **BitClout is not a company, and it doesn't have a board or CEO. Instead,
 it has a committed developer community that collaborates via this repo.**

## Determining What to Work On

If you would like to contribute to BitClout, we have several kanban boards that contain
ordered lists of the current priorities. The higher in the list, the
higher the priority. The boards are currently separated by task size and they are:
- [Small](https://github.com/orgs/bitclout/projects/3)
- [Medium](https://github.com/orgs/bitclout/projects/2)
- [Large](https://github.com/orgs/bitclout/projects/1)

We recommend new contributors start with small tasks and gradually work their way
up to large tasks. But this shouldn't discourage you from starting with a large task
if you're  really passionate about working on one of them. Just make sure you check
out the guidelines in the next section first.

## The Life of a Change

Changes can be small, medium, or large and there are different guidelines 
for each size of change. If you're unsure which guidelines to follow, simply 
[ask on Discord](https://discord.gg/bitclout) and someone will help you 
figure out the right path to submitting your change.

### Small Changes

Small changes do not require a CIP or extended discussion. Simply submit a pull request to the relevant repository
and you're done.

Examples of small changes include:
- Updating documentation
- Adding test coverage
- Fixing small bugs that don't affect consensus
- Correcting inconsistent APIs, logging, or configuration options
- Upgrading dependencies
- Clarify CIP process

### Medium Changes

For medium changes, please [create a new Discussion](https://github.com/bitclout/cips/discussions/new)
on the CIPs repository. You should explain the change you'd like to make, the rationale for making
the change, and how you plan to implement the change.

Once a [core dev](https://github.com/orgs/bitclout/people)
approves your idea you can get to work and open a pull request on the relevant repositories. You do not need to
open an issue or pull request on the CIPs repo.

Examples of medium changes include:
- Adding a new API endpoint
- Redesigning a frontend component
- Adding new configuration options
- Refactoring one package or a few files
- Change the CIP Process, excl those listed below under Large changes

### Large Changes

For large changes, a little more work is required. In order, you should:
1. [Create a new Discussion](https://github.com/bitclout/cips/discussions/new) on the CIPs repo
to get buy-in for your work from the community and at least one
[core dev](https://github.com/orgs/bitclout/people).
This is not always required. For example, if a core dev already approved your idea
then you can skip this step. Additionally, note that we are actively working to 
expand the group of core devs.
2. Once the community and at least one core dev approves your proposal, create a pull request
on the CIPs repo describing your change in full detail.
[Use the CIP template](https://github.com/bitclout/cips/blob/main/cip-template.md) for creating
your CIP.
3. The community and core devs will review your CIP PR and provide feedback. Once
at least two core devs approve, your PR can be merged and is now an official CIP.
4. Now that you have an official CIP, you can confidently submit pull requests to the relevant
repositories to implement your changes. You can also enlist other community or core developers to help you.
5. Once your PRs have at least two approvals from core devs, they can be merged and your
change can go live on the network.

Examples of large changes include:
- Changing or adding a transaction
- Any soft or hard fork to consensus
- Refactoring multiple packages or repositories
- A significant change to the CIPs process, including: 
  - Change levels & their definitions
  - Who approves changes at different levels

