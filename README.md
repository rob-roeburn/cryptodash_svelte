# CryptoDash

![alt text](https://github.com/rob-roeburn/cryptodash_svelte/blob/master/img/hero.png "Crypto Dash")

This is a small dashboard view to display the value of cryptocurrencies and record trading activity.  A portfolio view is displayed with position valuation.  Positions can be traded in and out, with a running total P&L displayed alongside the portfolio.

# Design

The app is single-page and uses the [Svelte](https://svelte.dev/). framework.  [AWS Lambdas](https://aws.amazon.com/lambda/) are deployed to allow serverless access to DynamoDB storage.  The lambda code is supplied in the main branch to allow deployment.

## Built With

* [Svelte](https://svelte.dev/) - The web framework used
* [NPM](https://www.npmjs.com/) - Package Management
* [AWS DynamoDB](https://aws.amazon.com/dynamodb/) - NoSQL data store
* [AWS Lambda](https://aws.amazon.com/lambda/) - Serverless API

## Authors

* **Rob Young**

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details
