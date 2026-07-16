- Feature Name: N/A
- Start Date: 2026-07-16
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: N/A

## Summary
[summary]: #summary

Adopt Python's AI policy as the AI policy for the Rust Project.

## Motivation
[motivation]: #motivation

In the Rust Project, we've seen an increase in unwanted and unhelpful contributions where contributors used generative AI.  These are frustrating and costly to reviewers in the Project.  We need to find ways to reduce the incidence of these and to lower the cost of handling them.

We hope that by stating our expectations clearly that fewer contributors will send us unhelpful things and more contributors will send us helpful ones.  We hope that this policy will make decisions and communication less costly for reviewers and moderators.

At the same time, the *process* of agreeing on an AI policy has been costly.  Project members have many reasonable and diverse views about generative AI and its appropriate role in society, in open source communities, and in the Project.  Many have struggled in trying to find an intersection of agreement.

Meanwhile, Python successfully [adopted] a [policy][python-policy] on generative AI.  Can we benefit from their work and adopt their result as our own?  Python is a project and community that is similar to ours in many ways.  If this can work for Python, can it work for us?  (And if not, why not?)

That's the question this request for comment humbly seeks to answer.

[python-policy]: https://devguide.python.org/getting-started/ai-tools/
[adopted]: https://github.com/python/devguide/pull/1778

## Guide-level explanation
[guide level explanation]: #guide-level-explanation

The [policy] below, adopted verbatim from Python's (modulo minor adjustment of Python-specific references; see the git history), is Rust's default policy on the use of AI tools when making contributions to the Rust Project.

## Reference-level explanation
[reference-level explanation]: #reference-level-explanation

The [policy] below is Rust's default policy on the use of AI tools when making contributions to the Rust Project.  Teams may still set their own policies (or may explicitly adopt this one).

This policy may be cited in or copied to all places where we guide contributors on our expectations.

## Guidelines for using AI tools
[policy]: #guidelines-for-using-ai-tools

The person submitting an issue or PR is responsible for its content, regardless of whether AI tools were used in its creation.  Generative AI tools can produce output quickly, but discretion, good judgment, and critical thinking are the foundation of all good contributions.  We value good code, concise accurate documentation, and well scoped PRs without unneeded code churn.

### Considerations for success

Authors must review the work done by AI tooling in detail to ensure it actually makes sense before proposing it as a PR or filing it as an issue.

We expect PR authors and those filing issues to be able to explain their proposed changes in their own words.

Disclosure of the use of AI tools in the PR description is appreciated, while not required.  Be prepared to explain how the tool was used and what changes it made.

Whether you are using AI tools or not, keep the following principles in mind for the quality of your contribution:

- Consider whether the change is necessary.
- Make minimal, focused changes.
- Follow existing coding style and patterns.
- Write tests that exercise the change.
- Keep our stability guarantees in mind.  Existing tests may be ensuring these are maintained.

Pay close attention to AI generated recommendations for testing changes.  Guide an AI model.  Always review the output before opening a pull request or issue, including proposed PR or issue titles and descriptions.

### Acceptable uses

Some of the acceptable uses of generative AI include:

- Assistance with writing comments, especially in a non-native language.
- Gaining understanding of existing code.
- Supplementing contributor knowledge for code, tests, and documentation.

### Unacceptable uses

Maintainers may close issues and PRs that are not useful or productive, regardless of whether AI tools were used or not.

If a contributor repeatedly opens unproductive issues or PRs, they may be blocked from contributing to the Project because it is disruptive and disrespectful of the maintainers time.

It is not acceptable to alter or bypass existing tests, or remove desired functionality, in order to make a failing test pass.  Such changes are not a real fix.

## Drawbacks
[drawbacks]: #drawbacks

In adopting any policy for contributions, we'll be asking more of our contributors.  On the other hand, this policy might help those contributors by providing clarity on what we expect.

This policy leaves a large gap between what's explicitly stated as acceptable and what's explicitly stated as unacceptable.  Within this gap, we'll all have to use good judgment, common sense, and trust.  But such a gap also may lead to uncertainty.

Some may view the adoption of a policy that does not go further in banning the use of generative AI as an implicit endorsement of the technology or of the companies currently working on it.  That's not what this is, but it's not always possible to prevent people from getting the wrong idea.

## Rationale and alternatives
[rationale and alternatives]: #rationale-and-alternatives

This policy encourages but does not *demand* disclosure of use.  Some feel strongly this should be demanded.  Others feel strongly this causes [harms].  This RFC adopts what Python adopted.

The policy suggests that assistance with writing comments is acceptable (especially for non-native speakers), but excessive use here can be particularly fraught.  This RFC adopts what Python adopted.

This policy serves as a default, but teams retain autonomy and can adopt other policies.  That preserves the ability of teams to fit their practices to their unique work and circumstances but it can make the Project feel more uneven.  This RFC errs on the side of team autonomy.

[harms]: https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5205850

## Prior art
[prior art]: #prior-art

Python adopted their version of this policy in [python/devguide#1778].  The Crates.io team adopted the Python policy, with similar minor adjustments, in [rust-lang/crates.io#13726].

Many other open-source projects have adopted various other policies, and within the Project, various other policies have been proposed or adopted.  In the interest of keeping this brief, we declined to attempt to exhaustively survey those here.

[python/devguide#1778]: https://github.com/python/devguide/pull/1778
[rust-lang/crates.io#13726]: https://github.com/rust-lang/crates.io/pull/13726

## Unresolved questions
[unresolved questions]: #unresolved-questions

None.

## Future possibilities
[future possibilities]: #future-possibilities

This RFC seeks to mitigate the harms in front of us (frustrated reviewers and Project members being burned out by the ongoing uncertainty and struggle to agree on a policy).  In the future, we may decide other things.  Maybe, on the basis of later experience, we will agree to ban generative AI entirely.  Or maybe we'll find better and more precise policies for living with it.  The world is moving quickly.  Let's adopt an intersection of agreement today and then see what happens later.
