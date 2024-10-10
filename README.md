# Better Fraud Prevention: Inform Sensitive Web Apps if They are Screen-Captured

## State of this document
The author of this document is a Google employee. However, at this time, this document only reflects the author’s opinions and suggestions. This document does NOT reflect Google’s official position; such a position would require an in-depth review by multiple stakeholders.

## Introduction
The Internet is rife with fraud. Scams often end up with honest people cheated out of their life savings. Sometimes this even leads victims to commit [suicide](https://edition.cnn.com/2024/06/17/asia/pig-butchering-scam-southeast-asia-dst-intl-hnk/index.html). Even extremely savvy people could be affected through their family members.

Any action we could take to reduce the prevalence of fraud would have a profound positive effect. In this case, our moral duty is fully aligned with our financial incentives, as all honest players stand to gain.
* **End-users** will keep their hard-earned fortunes.
* **Financial institutions** would lose less money to fraud (potentially passing on savings to customers).
* **Browser vendors** will protect their users better.
* **The Web platform** will be more attractive than otherwise.

This document focuses on two common types of fraud – the [technical support scam](https://en.wikipedia.org/wiki/Technical_support_scam) and the [overpayment scam](https://en.wikipedia.org/wiki/Overpayment_scam). While many variants exist, a common feature is the element of screen-capture. That is, the scammer often ends up on an audio call with the victim and convinces them to either:
* Install [remote administration](https://en.wikipedia.org/wiki/Remote_administration) software and give the scammer access.
* Share their screen using video conferencing software.

We propose a mechanism to render such schemes less viable.

## Goals and non-goals
### Goals
* Reduce the prevalence and severity of some specific types of scams, which are typically perpetrated on customers of financial institutions.
### Non-goals
* Complete elimination of fraud.
* Protect the employees of financial institutions.
* [Content protection](https://en.wikipedia.org/w/index.php?title=Content_protection&redirect=no) (as opposed to "user protection").

## Solution
### Core proposal
Web applications will be informed by the browser when they are affected by an ongoing screen-capture.
Only privileged Web applications will be able to read this signal. (Rationale follows further down.)

### API shape
As of this time, this document only seeks to align everyone on the overall characteristics of the solution. For illustrative purposes, a concrete proposal is presented, but the exact details are not important, so long as we agree on the overall direction.

```webidl
partial interface MediaDevices {
  readonly attribute boolean isScreenCaptured;
  attribute EventHandler onisscreencapturedchange;  // on-isScreenCaptured-change
};
```

With the concrete proposal above:
* `isScreenCaptured` reflects whether the captured application is visible in a tab, window or screen that is subject to screen-capture, either by another Web application or a native application.
* A blank event is fired on `onisscreencapturedchange` whenever the value of `isScreenCaptured` changes.

### Intended use
Financial institutions have extensive and robust systems for detecting suspicious activity and mitigating perceived risks. Examples of existing signals include:
* Sender’s – registered home address, location at transaction time, historical transactions
* Receiver’s – registered address, account age, historical transactions
* Time of day at sender’s and receiver’s locations
* Emerging patterns from recent fraudulent activity

Each new signal increases the ability of financial institutions to protect their users. We therefore aim to provide honest actors one more signal.

Once suspicious activity is detected, financial institutions employ a variety of mitigations, including delaying transactions until more verifications can be made. For example, they may call the end user on the next day and ensure that the transaction was not motivated by a scam. That is, financial institutions may go beyond changing the behavior of the Web application at the moment; they might even intentionally delay their actions, so that the fraudster would break off contact with the victim.

### Advantages
The proposed solution has several desirable properties.

First, financial institutions routinely deal with fraud. Their **expertise** in this domain far exceeds that possessed by browser vendors. The proposed solution keeps these more experienced actors in the driver’s seat, rather than a solution where the browser attempts to detect fraud itself.

Second, the proposed solution is **flexible**, in that different financial institutions may incorporate it differently into their fraud prevention ecosystem.

Third, the proposed solution is relatively **simple to implement** on platforms where the browser can receive this signal from the OS. On platforms where that is not possible, it’s nevertheless quite simple to implement the signal for capture by other Web applications, at least.

### Abuse potential
When a Web application detects that it is being captured, it can take action, and that action might not always be in the user’s interest. For example, applications might self-censor and prevent users from recording their interaction with the Web app, which would prevent a variety of use cases:
* Users might be prevented from recording tutorials.
* Users might be prevented from presenting slides without paying an extra fee.
* Users might be nudged or even forced to present using an affiliated video conferencing tool.
* Scam sites might limit users from seeking remote assistance from trusted relatives and friends. (Note that here, we are thinking of a different type of scams than those discussed earlier. Think for example of sites enticing users to divulge their credit card details.)

To avoid such abuse, we propose to only expose the signal to **allowlisted** Web applications. Entry into the allowlist will be subject to accepting terms and conditions which will include the promise to not self-censor or curtail the user’s actions. Failure to abide by these terms and conditions will result in expulsion from the allowlist.

### Problems with allowlists
An allowlist is used to mitigate the [abuse potential](#abuse-potential), because a better alternative was not found. We must nevertheless acknowledge the issues with such an approach.
* Places on browser vendors the onus of maintaining the allowlist.
* Potential legal liability when ejecting origins from the allowlist, or rejecting them to begin with.
* Potential divergence of allowlists between browsers.

### Technical details
#### Feasibility of detection by browser
Before a browser can inform the Web application of an event, the browser must first detect that event.

##### Capture by another Web application
It is trivial for browsers to detect when a sensitive Web application (SWA) is captured using tab-capture, since it is the browser itself that drives this capture. 

It is likewise trivial to detect when another Web application uses window-capture or screen-capture. (It could sometimes be challenging to detect whether the SWA is visible in that capture. However, the browser can err on the side of caution and assume that all screen-captures affect the SWA.)

##### Capture by a native application
The browser’s ability to detect an ongoing window-capture or screen-capture affecting the browser itself, driven by another native application, depends on the operating system and its configuration.

In the future, this document will be extended with an explanation of the state of the art. For the time being, suffice it to say that some operating systems allow this (e.g. iOS via [capturedDidChangeNotification](https://developer.apple.com/documentation/uikit/uiscreen/2921652-captureddidchangenotification)), and other operating systems could be extended to allow this.

If some operating systems harbor concerns about potential abuse, they may employ an allowlist method similar to that we employ ourselves; this is established practice – see for example macOS’ entitlements system.

#### Maintaining the allowlist
If this proposal is adopted and proves useful enough, we could explore whether the burden of maintaining the allowlist could be shared among browser vendors. Initially, we propose that each browser vendor implement their own allowlist.

In the case of Chromium, this could initially be implemented using the [Origin Trials mechanism](https://developer.chrome.com/docs/web-platform/origin-trials). Specifically, it could be a private origin trial, and a [Google Form](https://www.google.com/forms/about/) could be created for financial institutions to request a manually created token.

## Appendix I: Alternatives considered
### Alternative 0: Do nothing
Even with this signal, financial institutions would not be able to completely eliminate fraud. Would the effort taken here reduce the frequency and/or magnitude of fraud to a degree that justifies the effort? We argue that yes; however, reasonable minds could differ.

### Alternative 1: Prevent capture
It has previously been proposed that Web applications should be able to opt out of being captured. One such proposal was that Web applications should be able to mark themselves as uncapturable, and that the browser and OS should then ensure that these Web applications are not captured, either by blanking them out from active screen-captures, or by preventing these screen-captures to begin with.

The main problem with this approach is that it takes away control from the user and prevents legitimate use of screen-sharing, often to the user’s detriment. This is true both for the general case ("why should I not be able to record a tutorial of my use of Figma?"), as well as for the financial institution case ("my spouse is away on a business trip, why can’t we discuss our financials over a call?").

In other words - while it might solve the fraud problem, it would do so at the cost of suppressing legitimate use of screen-capture.

### Alternative 2: Warn the user
We could introduce a mechanism for Web applications to declare themselves as sensitive, by specifying a new [boolean attribute](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#boolean-attribute) called "sensitive" that applies to the `<head>`, `<body>` or `<html>` tags.
Before initiating any new screen-capture sessions by another Web application, if that capture would affect a sensitive Web application, the browser would warn the user and possibly also ask them to reconfirm their intention.
When the browser detects that a native application starts a screen-capture session, it could temporarily hide sensitive Web applications, warn the user and ask for additional confirmation before unhiding sensitive Web applications.
When the browser is aware of an ongoing screen-capture, either by a native application or a Web application, it would warn the user and possibly ask them for additional confirmation, before loading sensitive Web applications, either through navigation, changing of the active tab, etc.

The main problem with this approach is that the scammer can dismiss such warnings or convince the user to ignore them. This is especially true when the scam is driven by [remote administration](https://en.wikipedia.org/wiki/Remote_administration) software, in which case the scammer can rearrange content on the screen to obscure crucial information from the victim.

### Alternative 3: Enterprise policies and/or employees
We seek to protect financial institutions’ customers, not their employees. We cannot expect financial institutions to have the power to set policies or force-install extensions on customers’ machines.

(It bears mentioning that many enterprises do wish to limit screen-sharing on their employees’ devices. On Chromium-based browsers, that is already possible using enterprise policies today. Further policies may be added independently of the effort described by this document.)

### Alternative 4: Extension APIs
As mentioned in [alternative 3](#alternative-3-enterprise-policies-andor-employees), we cannot expect financial institutions to force-install extensions on their customers’ devices. But could we expect them to convince their customers to do so…?

The author of this document would argue that:
Mayn users would not end up installing such an extension; possibly most of them won’t.
It would be detrimental to security on the Web, if users were taught that it’s legitimate for a website to plead with them to install an extension.

## Appendix II: Potential future extensions
It **might** also be useful for Web applications to know:
* Whether the capture is associated with [remote administration](https://en.wikipedia.org/wiki/Remote_administration) software or just a vanilla capture, likely associated with a video conferencing application or screenshotting tool.
* Whether the capture is part of some trusted application or module of the OS itself, especially if those become commonplace in the context of AI-driven tools. (For instance, [Windows Recall](https://support.microsoft.com/en-us/windows/retrace-your-steps-with-recall-aa03f8a0-a78b-4b3e-b0a1-2eb8ac48701c).) 

## Appendix III: Q&A
##### "What if the browser is running fully under remote admin? (CRD or Citrix, for instance.) Does that count as capture?"
It is not obvious that the browser could distinguish this case from other screen-capture scenarios.

But even if it’s possible, is it desirable? The author would argue that this should not be treated differently, or else attackers could orchestrate this scenario and circumvent the signal.

##### "Allowlists are antithetical to the open Web."
First – yes, allowlists are not ideal. If a better alternative could be suggested, we could pivot to that. But for the time being, this appears to be the best way to handle the trade off between users’ safety on the Web and the abuse potential of exposing the signal to all Web applications.

Second – note that failure to appear on the allowlist proposed here, leaves Web applications in their current status quo. The author of this document believes that there is a fundamental difference between hiding core functionality behind an allowlist, and exposing "extra" information behind an allowlist.

##### "What about partially visible applications?"
We report to the application that it is captured, even if a single of its pixels is captured.

Further, if we ever know that something is captured, but technical limitations prevent us from knowing for certain whether the application is affected, we will err on the side of caution and assume that it is.

This is reasonable, because screen-capture is common, but not the norm. Most applications, during most sessions, are not captured; most of the time, nothing is captured on the system.

##### "Should the user have a way to override?"
No, because any override the user can reach, the scammer can also reach. (Recall that the scammer either has remote access to the machine, and/or they have power to manipulate the user into taking action on their behalf.)
