---
title: Triggers and Payloads
parent: Building Your Quest
---

# Triggers and Payloads

System triggers can generate a payload that gets passed to `action` and `condition` blocks along with the global payload. When defining a call to an action/condition block in the flow node, developers specify how to map a parameter from the payload to a parameter passed to the block. 

This document:

- Lists the available triggers along with the payload generated by each of them.
- Lists the global payload parameters.
- Explains how the payload is mapped to actions/conditions blocks parameters.
- Explains the usage of the special `result` parameter passed to each condition and action.

## Triggers

### USER_MESSAGE

Triggered when the user sends a chat message to one of the bots.
This trigger must contain `params` with a `person` key.
For example:

```yaml
trigger: 
  type: user_message
  params:
    person: keen
  flowNode:
    ...
```

Generated payload:

- `userMessageText`: the text the user wrote to the bot.

### GITHUB_PR_LIFECYCLE_STATUS

Triggered when the status of a PR opened by the user is changed.

Generated payload:

- `eventType:` the status of the PR. Equal to one of the following:
    - `github_pr_opened:` PR opened by the user
    - `github_pr_workflow_complete_success:` All PR checks passed successfully
    - `github_pr_workflow_complete_failure:` One of the PR checks failed
    - `github_pr_merged:` User merged the PR
- `githubPrNumber:` PR number
- `githubRepository:` The name of the repository in which the PR was opened

### PING

Triggered when a ping event happens in the user’s Anythink repo. No specific payload.

### CHAT_OPENED

Triggered when the user opens Snack. No specific payload.

### LOCAL_PAGE_LOAD

Triggered when the user opens a page in the Anythink frontend.

Payload:

- `path:` The URL of the opened page.

### HEROKU_RELEASE_CREATED

Triggered when a new Heroku release is created. 

Payload:

- `eventType:` type of the event that caused new release. For example, when setting config var: `^Set (.*) config var`
- `eventData:` The full body sent by Heroku. See more example at [Heroku's webhook event documentation].

### GITHUB_USER_CONNECTED

Triggered when the user accepts the invitation to his Github repo. No specific payload

### NEWRELIC_EVENT_RECEIVED

Triggered when an event is received from the user’s New Relic account.

### CHAT_FORM_SUBMITTED
Triggers when a a user submit a form on Snack.

Payload:

- `formSubmission`: The data submitted by the user. The type is different between form types -
  - `single_select_form` - a single string
  - `multi_select_form` - a list of strings

Params: 

- `formId`: Used to subscribe to the desired form.

Usage example (for single select form):

```yml
trigger:
  type: chat_form_submitted
  params:
    formId: your_single_select_form_id
  flowNode:
    if:
      conditions:
      - conditionId: text_contains_strings
        params:
          text: "${formSubmission}"
          strings:
          - accept
      then:
        do:
        - actionId: your action here
      else:
        do:
        - actionId: a different action here
```


## Global Payload

In addition to specific payload passed by each trigger, a global payload is always available:

- `user`: The Wilco user object. Contains the following fields (partial list):
    - `firstName` User's first name
    - `lastName` User's last name
    - `email` User's email address
    - `githubuser:` User’s Github username
    - `repository:` User’s Github repo url
    - `repoName:` Name of the user’s Github repo
    - `company`
    - `backendHerokuAppName:` the name of the backend Heroku app
    - `frontendHerokuAppName:` the name of the frontend New Relic app
    - `newrelic:` User’s New Relic account information
        - `accountId` New Relic account id
        - `apiKey` New Relic API key
    - `frameworks:` The backend and frontend frameworks chosen by the user
        - `backend:` One of node/rails/pythob
        - `frontend:` Currently only react
        - `database:` One of mongodb/postgresql
    
    Usage example:
    
    ```yaml
    actionId: github_pr_approve
    params:
      person: keen
      message: Nailed it! Excellent job @**${user.githubuser}**! You can now merge the PR.
    ```
    
- `bots:` Info about the bots. A map from bot id to bot details.
    
     Available ids are:

    - `keen`
    - `lucca`
    
    information for each bot:
    
    - `displayName:` name of the bot. E.g, Ness or Navi
    
    Usage example:
    
    ```yaml
    actionId: bot_message
    params:
      person: lucca
      messages:
      - text: Hey ${user.firstName},
        delay: 2000
      - text: I'm **${bots['lucca'].displayName}**, the lucca team leader.
        delay: 1500
      - text: "**${bots['keen'].displayName}** asked me to help you out. Just write
          here your *GitHub username* (without the \\`@\\`, so Messenger doesn’t freak
          out), and we'll add you as a contributor to the company's repository."
        delay: 3500
    ```
    
- `nextQuestUrl:` url for the next quest.
    
    Equal to: `${CONFIG.ENVIRONMENT.WILCO_APP_URL}/home`
    

## Mapping payload to block params

- All payload (trigger + global) parameters are passed as is to each block. This way, no additional configuration in the YAML is required for trivial cases.
    
    A good example is Github blocks which usually require a pr number or repo name:
    
    ```jsx
    const isFileInPRContains = async ({
      user,
      githubPrNumber,
      fileName,
      regex,
    }) => {
      const file = await getAddedFile(user, githubPrNumber, [fileName]);
      return !!file?.patch?.match(RegExp(regex));
    };
    ```
    
    In this example,  `user` and `githubPrNumber` are available in the condition parameters without any further configuration. Calling the `github_is_file_contains` can be configured in the YAML file, for example, in the following way (only `regex` and `fileName` are defined):
    
    ```yaml
    if:
      conditions:
        - conditionId: github_is_file_contains
          params:
            regex: license_key
          paramsFramework:
            node:
              fileName: backend/newrelic.js
            rails:
              fileName: backend/config/newrelic.yml
            python:
              fileName: backend/newrelic.ini
      then:
        ...
    ```
    

- In many cases, it is required to map the payload parameter to the block parameter. Referencing payload parameters is done using `${<payload_param>}`. Two such cases can be:
    - The name of the block parameter is different from its name in the payload
        
        An example of such a case is checking if chat message matches a regex:
        
        ```yaml
        if:
          conditions:
          - conditionId: text_match_regex
            params:
              text: "${userMessageText}"
              regex: ".*localhost.*@"
          then:
        ```
        
        In this example, the param `userMessageText` is passed to the condition as `text`
        
    - It is required to pass the payload parameter after some manipulation or as part of a more complex string (or both). 
    
    An example for such a case:

    ```yaml
        do:
          - actionId: bot_message
            params:
              person: keen
              messages:
              - text: Ooohh, I didn’t expect you to do this so quickly! Even your username,
                  ${path.slice(2)}, brings back memories. Strangely enough, they’re
                  your memories, which I don’t know why I have.
                delay: 2000
        ```
        
        We embed `path` param from the payload to the text sent to `keen` bot. The param here is embedded after applying `slice(2)` on its value.
        

## Result parameter

In addition to the payload parameters, a `result` parameter is passed to all conditions and actions. Each block can decide if to use it or not. The `result` parameter is used to pass information from the block to the next blocks. 

- The result is unique for each block and they are not shared, which means that a block can’t override or change the result object of other blocks.
- The result will be stored only if `name` was set for the block. In that case, the result of a block named `some_name` will be stored in the global payload in `payload.outputs[some_name]`. If name is missing, the result won't be stored and won't be used.
    
    A more specific example:
    
    ```yaml
    conditions:
      - conditionId: heroku_check_backend_config_var
        name: new_relic_license_key_config
        params:
          key: NEW_RELIC_LICENSE_KEY
    then: ...
    ```
    
    In this example, we call an action with the id `heroku_check_backend_config_var` and give it a name,  `new_relic_license_key_config.` This condition check that a config var exists, and also sets its value on the result, if exists. 
    
    ```json
    {
    	"value": "config_var_val"
    }
    ```
    
    Then, the config var value can be used in next blocks. Foe example:
    
    ```yaml
    if:
      conditions:
        - conditionId: newrelic_license_key_valid
          params:
            licenseKey: '${outputs.new_relic_license_key_config.value}'
      then: ...
    ```
    
    The config var is used as input to the `newrelic_license_key_valid` condition which checks that the value is a valid New-Relic license key.
    

A default result is initialized for all conditions and actions.

- For actions, the result is initialized to:
    
    ```json
    {
    	"success": true
    	"error": null
    }
    ```
    

- For conditions, the result is initialized to:
    
    ```json
    {
    	"error": null
    }
    ```
    
    There's no need for `success` as conditions always return true/false and `success` is set according to the result. Meaning, `output.condition_name` will contain `success` as well.

[Heroku's webhook event documentation]: https://devcenter.heroku.com/articles/webhook-events#api-release