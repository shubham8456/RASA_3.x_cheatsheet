# RASA 3.x

----

Rasa Open Source is a framework for Natural Language Understanding (NLU), dialogue management, and integrations.
<br/>
[RASA 3.x Open Source Documentation](https://rasa.com/docs/rasa/)

----

### Setting up local environment
https://rasa.com/docs/rasa/installation/environment-set-up

----

### RASA cheatsheet to get quick overview
https://chatbotslife.com/the-ultimate-rasa-cheatsheet-925f8980d51b

Get high level overview of the following concepts:
- file structure of a common RASA project
- CDD (Conversation Driven Development) philosophy ([read more...](https://rasa.com/docs/rasa/conversation-driven-development/))
- RASA CLI and some important commands

<details>
<summary>Click to expand cheatsheets</summary>

![RASA Cheatsheet 1](/images/rasa_cheatsheet_01.png)
![RASA Cheatsheet 2](/images/rasa_cheatsheet_02.png)
</details>

----

## Best Practices

### - Generating NLU Data

Resource: https://rasa.com/docs/rasa/training-data-format/

<details>
<summary>Click to expand</summary>

- **What is NLU? :** Natural Language Understanding is the part of Rasa that performs intent classification, entity extraction, and response retrieval.

- NLU will take in a sentence such as "I am looking for a French restaurant in the center of town" and return structured data like:
```json
{
  "intent": "search_restaurant",
  "entities": {
    "cuisine": "French",
    "location": "center"
  }
}
```

- **User Messages:** All user messages are specified with the `intent:` key and an optional `entities:` key.

- Improving entity recognition in Rasa can be done using pre-trained entity extractors, regexes, lookup tables, and synonyms.

- Handling edge cases like misspellings and defining an out-of-scope intent are important strategies for robust NLU models.

- **Defining an Out-of-scope Intent:** It is always a good idea to define an out_of_scope intent in your bot to capture any user messages outside of your bot's domain. When an out_of_scope intent is identified, you can respond with messages such as "I'm not sure how to handle that.".

- Adding a character-level featurizer provides an effective defense against spelling errors by accounting for parts of words, instead of only whole words. You can add character level featurization to your pipeline by using the char_wb analyzer for the CountVectorsFeaturizer, for example:
```yml
pipeline:
# <other components>
- name: CountVectorsFeaturizer
  analyze: char_wb
  min_ngram: 1
  max_ngram: 4
# <other components>
```

#### Avoiding Intent Confusion

Intent confusion often occurs when you want your assistant's response to be conditioned on information provided by the user. For example, "How do I migrate to Rasa from IBM Watson?" versus "I want to migrate from Dialogflow."

Since each of these messages will lead to a different response, your initial approach might be to create separate intents for each migration type, e.g. `watson_migration` and `dialogflow_migration`. However, these intents are trying to achieve the same goal (migrating to Rasa) and will likely be phrased similarly, which may cause the model to confuse these intents.

To avoid intent confusion, group these training examples into single `migration` intent and make the response depend on the value of a categorical `product` slot that comes from an entity. This also makes it easy to handle the case when no entity is provided, e.g. "How do I migrate to Rasa?" For example:

```yml
stories:
- story: migrate from IBM Watson
  steps:
    - intent: migration
      entities:
      - product
    - slot_was_set:
      - product: Watson
    - action: utter_watson_migration

- story: migrate from Dialogflow
  steps:
    - intent: migration
      entities:
      - product
    - slot_was_set:
      - product: Dialogflow
    - action: utter_dialogflow_migration

- story: migrate from unspecified
  steps:
    - intent: migration
    - action: utter_ask_migration_product
```

</details>

----

### - Writing Conversation Data

Resource: https://rasa.com/docs/rasa/writing-stories

This includes the stories and rules that make up the training data for your Rasa assistant's dialogue management model. Well-written conversation data allows your assistant to reliably follow conversation paths you've laid out and generalize to unexpected paths.

#### 1. When to Write Stories vs. Rules

<details>
<summary>Click to expand</summary>

Rules are a type of training data used by the dialogue manager for handling pieces of conversations that should always follow the same path. Rules can be useful when implementing:

1. One-turn interactions: Some messages do not require any context to answer them. Rules are an easy way to map intents to responses, specifying fixed answers to these messages.

2. Fallback behavior: In combination with the FallbackClassifier, you can write rules to respond to low-confidence user messages with a certain fallback behavior.

3. Forms: Both activating and submitting a form will often follow a fixed path. You can also write rules to handle unexpected input during a form.

</details>

#### 2. When to Use Slots to Influence Conversations

<details>
<summary>Click to expand</summary>

- Slots act as your bot’s memory. When you define a slot, you can define whether a slot should influence the conversation or not. Slots with the property `influence_conversation` set to false can only store information. Slots with the property `influence_conversation` set to true can affect the dialogue flow based on the information stored in it.

- All slots which influence the conversation need to be added to your stories or rules.

- For example, you can use a boolean slot set by a custom action to control the dialogue flow based on its value using the following stories:
```yaml
stories:
- story: Welcome message, premium user
  steps:
   - intent: greet
   - action: action_check_profile
   - slot_was_set:
     - premium_account: true
   - action: utter_welcome_premium

- story: Welcome message, basic user
  steps:
   - intent: greet
   - action: action_check_profile
   - slot_was_set:
     - premium_account: false
   - action: utter_welcome_basic
   - action: utter_ask_upgrade
```

</details>

#### 3. Implementing Branching Logic

<details>
<summary>Click to expand</summary>

- In cases where a value is used only to determine the bot's response, consider embedding the decision logic inside a custom action as opposed to using a featurized slot in your stories. This can help reduce overall complexity and make your stories easier to manage.

- For example, you can convert these stories:
```yaml
stories:
- story: It's raining now
  steps:
  - intent: check_for_rain
  - action: action_check_for_rain
  - slot_was_set:
    - raining: true
  - action: utter_is_raining
  - action: utter_bring_umbrella

- story: It isn't raining now
  steps:
  - intent: check_for_rain
  - action: action_check_for_rain
  - slot_was_set:
    - raining: false
  - action: utter_not_raining
  - action: utter_no_umbrella_needed
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;into a single story:
```yaml
stories:
- story: check for rain
  steps:
  - intent: check_for_rain
  - action: action_check_for_rain
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;with the custom action code:
```python
def run(self, dispatcher, tracker, domain):
    is_raining = check_rain()
    if is_raining:
        dispatcher.utter_message(template="utter_is_raining")
        dispatcher.utter_message(template="utter_bring_umbrella")
    else:
        dispatcher.utter_message(template="utter_not_raining")
        dispatcher.utter_message(template="utter_no_umbrella_needed")
    return []
```

- In cases where the value is used to influence the action flow going forward, return a featurized slot to determine the stories. For example, if you want to collect information about new users, but not returning ones, your stories might look like this:

```yaml
stories:
- story: greet new user
  steps:
  - intent: greet
  - action: check_user_status
  - slot_was_set:
    - new_user: true
  - action: utter_greet
  - action: new_user_form
  - active_loop: new_user_form
  - active_loop: null

- story: greet returning user
  steps:
  - intent: greet
  - action: check_user_status
  - slot_was_set:
    - new_user: false
  - action: utter_greet
  - action: utter_how_can_help
```
</details>

#### 4. Using OR statements

<details>
<summary>Click to expand</summary>

- In stories where different intents or slot events are handled by your bot in the same way, you can use OR statements as an alternative to creating a new story.

- For example, you can merge these two stories:
```yaml
stories:
- story: newsletter signup
  steps:
  - intent: signup_newsletter
  - action: utter_ask_confirm_signup
  - intent: affirm
  - action: action_signup_newsletter

- story: newsletter signup, confirm via thanks
  steps:
  - intent: signup_newsletter
  - action: utter_ask_confirm_signup
  - intent: thanks
  - action: action_signup_newsletter
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;into a single story with an OR statement:
```yaml
stories:
- story: newsletter signup with OR
  steps:
  - intent: signup_newsletter
  - action: utter_ask_confirm_signup
  - or:
    - intent: affirm
    - intent: thanks
  - action: action_signup_newsletter
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;At training time, this story will be split into two original stories.
</details>

#### 5. Using Checkpoints

<details>
<summary>Click to expand</summary>

- Checkpoints are useful for modularizing your stories into separate blocks that are repeated often. For example, if you want your bot to ask for user feedback at the end of each conversation flow, you can use a checkpoint to avoid having to include the feedback interaction at the end of each story:
```yaml
stories:
- story: beginning of conversation
  steps:
  - intent: greet
  - action: utter_greet
  - intent: goodbye
  - action: utter_goodbye
  - checkpoint: ask_feedback

- story: user provides feedback
  steps:
  - checkpoint: ask_feedback
  - action: utter_ask_feedback
  - intent: inform
  - action: utter_thank_you
  - action: utter_anything_else

- story: user doesn't have feedback
  steps:
  - checkpoint: ask_feedback
  - action: utter_ask_feedback
  - intent: deny
  - action: utter_no_problem
  - action: utter_anything_else
```
**DO NOT OVERUSE:** Using checkpoints inside existing checkpoints is highly discouraged as this increases training time significantly and makes your stories difficult to understand.
</details>

#### 6. Creating Logical Breaks in Stories

<details>
<summary>Click to expand</summary>

<br/>
Always consider separating longer stories into smaller conversational blocks that will handle the sub-tasks.

A happy path story for handling a lost credit card might look like:
<details>
<summary>Click to expand example</summary>

```yaml
stories:
- story: Customer loses a credit card, reviews transactions, and gets a new card
  steps:
  - intent: card_lost
  - action: check_transactions
  - slot_was_set:
    - reviewed_transactions: ["starbucks"]
  - action: utter_ask_fraudulent_transactions
  - intent: inform
  - action: action_update_transactions
  - intent: affirm
  - action: utter_confirm_transaction_dispute
  - action: utter_replace_card
  - action: mailing_address_form
  - active_loop: mailing_address
  - active_loop: null
  - action: utter_sent_replacement
  - action: utter_anything_else
  - intent: affirm
  - action: utter_help
```
</details>
<br/>
This involves a series of sub-tasks, namely checking spending history for fraudulent transactions, confirming a mailing address for a replacement card, and then following up with the user with any additional requests. In this conversation arc, there are several places where the bot prompts for user input, creating branching paths that need to be accounted for.
<br/><br/>
For example, when prompted with "utter_ask_fraudulent_transactions", the user might respond with a "deny" intent if none are applicable. The user might also choose to respond with a "deny" intent when asked if there's anything else the bot can help them with.
<br/><br/>
We can separate out this long story into several smaller stories as:

```yaml
stories:
- story: Customer loses a credit card
  steps:
  - intent: card_lost
  - action: utter_card_locked
  - action: spending_history_form
  - active_loop: spending_history_form
  - active_loop: null
  - slot_was_set:
    - reviewed_transactions: ["starbucks"]
  - action: utter_ask_fraudulent_transactions

- story: Customer reviews transactions and gets a new card
  steps:
  - action: utter_ask_fraudulent_transactions
  - intent: inform
  - action: action_update_transactions
  - slot_was_set:
    - reviewed_transactions: ["target", "starbucks"]
  - intent: affirm
  - action: utter_confirm_transaction_dispute
  - action: utter_replace_card
  - action: mailing_address_form
  - active_loop: mailing_address
  - active_loop: null
  - action: utter_sent_replacement
  - action: utter_anything_else

- story: Customer has something else they need help with
  steps:
  - action: utter_anything_else
  - intent: affirm
  - action: utter_help
```
</details>

#### 7. Handling Context Switching
https://rasa.com/docs/rasa/writing-stories#handling-context-switching
<details>
<summary>Click to expand</summary>

- Often, users will not respond with the information you ask of them and instead deviate from the happy path with unrelated questions. Using CDD to understand what unhappy paths your users are taking, you can create stories for handling context switching.

- You can use either of Rules and Stories for context switching as per the requirement

- Example of context switching using Stories, if you want to switch context upon a user ask and then return to the original flow after that ask is complete:
![Example of Context Switching using Stories](/images/context_switching_using_stories.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;You will need to create a story that describes this context-switching interaction:

```yaml
stories:
- story: Context switch from credit card payment to money transfer
  steps:
  - intent: pay_credit_card
  - action: credit_card_payment_form
  - active_loop: credit_card_payment_form
  - intent: transfer_money                         # - user requests a money transfer
  - active_loop: null                              # - deactivate the credit card form
  - action: transfer_money_form                    # - switch to the money transfer form
  - active_loop: transfer_money_form
  - active_loop: null
  - action: utter_continue_credit_card_payment     # - once the money transfer is completed,
                                                   #   ask the user to return to the
                                                   #   credit card payment form
```
</details>

----

### - Managing Conversation Data Files

When writing stories and rules, it's usually a good idea to create separate files based on the types of conversations being represented.

For example, you might create a file `chitchat.yml` for handling chitchat, and a `faqs.yml` file for FAQs. Refer to the [rasa-demo bot](https://github.com/RasaHQ/rasa-demo) for examples of story file management in complex assistants.

----

### - Using Interactive Learning
https://rasa.com/docs/rasa/writing-stories/#using-interactive-learning

----

