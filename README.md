## K6 <> GitHub PR Comments

This library automatically creates a report from K6 execution, and publishes it as a GitHub comment to your Pull Request.

Thanks to [Kamil Kisiela](https://github.com/kamilkisiela/) for the initial script.

![example](https://i.imgur.com/3y7c957.png)

Features:
* Easy integration with K6 CLI
* Automatically creates/updates existing comments.
* Flexible output, and flexible results.
* Full report log attached to the comment.

### Getting Started

1. Make sure you have GitHub personal access token available. If you are using GitHub Actions, you can re-use the one created automatically. If not, [create one here](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).
2. Configure your K6 to run in your CI, and configure the K6 script the way you wish. 
3. Import the script directly from this repo, into your K6 code, and add `handleSummary` function to use it:

```js
import { check } from "k6";
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.1/index.js";
import { githubComment } from "https://raw.githubusercontent.com/dotansimha/k6-github-pr-comment/master/lib.js";

export const options = {
  // Here are all your K6 config and options.
  // It's useful to add `thresholds` here, since you can alter choose to fail the PR based on that.
};

// Add this function and make sure it's exported
export function handleSummary(data) {
  githubComment(data, {
    token: __ENV.GITHUB_TOKEN, 
    commit: __ENV.GITHUB_SHA,
    pr: __ENV.GITHUB_PR,
    org: "my-username-or-org",
    repo: "repo-name",
    renderTitle({ passes }) {
      return passes ? "✅ Benchmark Results" : "❌ Benchmark Failed"; // Here you can choose how to build the title
    },
    renderMessage({ passes, checks, thresholds }) { // Customize the output and the comment text
      const result = [];

      // In case of failures in thresholds, you can customize the message
      if (thresholds.failures) {
        result.push(
          `**Performance regression detected**: it seems like your Pull Request adds some extra latency to the GraphQL requests, or to envelop runtime.`
        );
      }

      // In case of failing execution of K6, you can customize the output
      if (checks.failures) {
        result.push("**Failed assertions detected**: some GraphQL operations included in the loadtest are failing.");
      }

      if (!passes) {
        result.push(`> If the performance regression is expected, please increase the failing threshold.`);
      }

      // Make sure to return a string
      return result.join("\n");
    },
  });

  // This will preserve the original output of K6 to the console, while still publishing to GH.
  return {
    stdout: textSummary(data, { indent: " ", enableColors: true }),
  };
};
```

Now, make sure to pass the required environment variables. If you are using GitHub Actions, this can be done like that (while running from a GH workflow): 

```
k6 -e GITHUB_PR=${{ github.event.number }} -e GITHUB_SHA=${{ github.sha }} -e GITHUB_TOKEN=${{ secrets.GITHUB_TOKEN }} run ./my-k6-file.js 
```