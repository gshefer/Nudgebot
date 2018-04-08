# Nudgebot

Nudgebot is a generic Github repositories tracker bot which allows to efficiently automate, manage and monitor the organization github repositories.

It provides you the ability to model the statistics you want to collect from the repositories and execute tasks by specific rules.
The library includes several third party libraries like IRC, Google calendar, Github (and so on in the future) which could also be used for analysis and tasks.

## Getting started

### Prerequisits
1. [Python-3.6.4](https://www.python.org/downloads/release/python-364/) and above.
2. [RabbitMQ](https://www.rabbitmq.com/) - for the celery runner.
3. [MongoDB](https://www.mongodb.com).

### Setup a project
1. _Start a project._
    - ```python manage.py startproject <name_or_path>```
    - *For more description run: ```python manage.py -h```
    - Once the project directory has been created, it has the following structure:
    ```
    .
	|-- config                           # This is the configuration directory
	|   |-- __init__.py
	|   |-- config.template.yaml         # The main config file, it includes the main global configurations.
	|   |-- credentials.template.yaml    # The credential configuration file, it includes all the credentials. data.
	|   `-- users.template.yaml          # The users configuration file, it includes all the users data (contact info for every channel).
	|-- statistics                       # The statistics directory, here you should create your statistics classes.
	|   `-- __init__.py
	`-- tasks                            # The tasks directory, here you should create your tasks classes.
	    `-- __init__.py
	|-- __init__.py
	|-- assets.py                        # This file includes all the assets for the project (The DB client, The Bot, The celery runner, etc.)
	|-- main.py                          # The main file. This file you should run to actually run the bot.

	4 directories, 8 files
    ```
2. _Configure your project._
    - Create your configuration files from the templates inside the `./config` directory.
3. _Create your [statistics class](https://github.com/gshefer/Nudgebot/blob/master/nudgebot/statistics/base.py)es inside the statistics package in the project._
    - Example:
      ```python
      from nudgebot.statistics.base import statistic
      from nudgebot.statistics.github import PullRequestStatistics
      
      class MyPrStatistics(PullRequestStatistics):
      """In this statistics class we collect all the statistics that related to pull request."""
      key = 'my_pr_stats'  # This key will be used to access this statistics in the tasks

      # We decorate this getter with `statistic` decorator to indicate that this
      # is a statistic that we would like to collect and save
      @statistic
      def total_number_of_comments(self):
          return self.party_scope.comments

      ```
    - [Example for statistics classes](https://github.com/gshefer/Nudgebot/blob/master/examples/project_a/statistics/__init__.py) could be found in the [example projects](https://github.com/gshefer/Nudgebot/tree/master/examples/project_a).
4. _After you happy with your statistics. Create the [tasks](https://github.com/gshefer/Nudgebot/blob/master/nudgebot/tasks/base.py) inside the tasks package._
    - Example:
      ```python
      from nudgebot.tasks import ConditionalTask
      from nudgebot.thirdparty.github.base import Github
      from nudgebot.thirdparty.github.pull_request import PullRequest
      from nudgebot.thirdparty.irc.base import IRCparty


      class PromptWhenLargeNumberOfComments(ConditionalTask):
          """This task is prompting on IRC when there is a large number of comment in a pull request."""

          Party = Github()                            # The third party for this task is Github.
          PartyScope = PullRequest                    # The scope of this task is pull request.
          NAME = 'PromptWhenLargeNumberOfComments'    # The name of the task.
          PR_MAX_NUMBER_OF_COMMENTS = 10

          @property
          def condition(self):
              # Checking that total number of comment is greater than `PR_MAX_NUMBER_OF_COMMENTS`.
              return self.statistics.my_pr_stats.total_number_of_comments > self.PR_MAX_NUMBER_OF_COMMENTS

          def get_artifacts(self):
              return [str(self.statistics.my_pr_stats.total_number_of_comments)]

          def run(self):
              # Running the task
              IRCparty().client.msg(
                  '##bot-testing',
                  f'PR#{self.statistics.my_pr_stats.number} has more than {self.PR_MAX_NUMBER_OF_COMMENTS} comments! '
                  f'({self.statistics.my_pr_stats.total_number_of_comments} comments)'
              )

      ```
    - [Example for task classes](https://github.com/gshefer/Nudgebot/blob/master/examples/project_a/tasks/__init__.py) could be found in the [example projects](https://github.com/gshefer/Nudgebot/tree/master/examples/project_a).


### Running the project
After you configured your statistics and tasks, only you have to do is just run the `main.py` file inside the project root(```python main.py```).
Once you run it, it'll run the celery app in the background, creating an initial poll and then will run the bot's main loop.


---

#### Notes:
1. _The example project are great knowledge source and great guides for start_
2. _There was an old designed of nudgebot which is less generic, but this one is based on it:_ [https://github.com/gshefer/NudgeBot-old](https://github.com/gshefer/NudgeBot-old)
