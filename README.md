# factorio-server

Terraform configuration for a private Factorio server on AWS, shared by two players. Either player can start the server alone, play the shared world alone or together, and stop the server. The server is an EC2 instance that runs only while someone plays and stops itself after 15 minutes without players, so AWS bills compute time only during play.

See [PLAN.md](PLAN.md) for the build plan, the design decisions, and progress.
