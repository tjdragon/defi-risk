# DeFi Risk

March, 21st 2023

## Intro

Decentralized Finance, or DeFi, centers around empowering transactions, exchanges, and financial services based on digital assets.  
The fundamental principle of DeFi is the absence of a centralized authority to govern or manipulate operations.  
In the context of financial crime risk, this brief article proposes an approach to evaluating DeFi while remaining mindful of your risk appetite.

## Token risk

We will first start by looking at token risk (from a financial crime lens).  
For the rest of this article, tokens relate to ERC20 tokens on the Ethereum ecosystem.

### An ERC20 primer
The [ERC20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) standard does not allow to list
all its holders, meaning it is possible to say how many tokens an address holds ('balanceOf'), but it is not possible
using the standard ERC20 functions to list all the holders of a given token.  
For this, we need to rely on the ['transfer' events](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-Transfer-address-address-uint256-) 
and use an archive node to be able to get all the incoming and outgoing transfers.  

### Querying the data: list all holders
There are a couple of ways to do this, the DIY version, the use of [TheGraph](https://thegraph.com/) which
is not the faint-hearted, and the one I have used, the lazy but very powerful and efficient option: [GCP](https://cloud.google.com/).  

Google has made several blockchain databases available that can be queried using [BigQuery](https://cloud.google.com/bigquery).  

#### Example 1: Top 10 ETH addresses by balance
The following query returns the top 10 addresses by balance on Ethereum:

```sql
select *
from `bigquery-public-data.crypto_ethereum.balances`
order by eth_balance desc
limit 10
```

This query processes 13.82 GB of data at this time of writing - and took 1 second to run.  
The first address being returned is [0x00000000219ab540356cbb839cbe05303d7705fa](https://etherscan.io/address/0x00000000219ab540356cbb839cbe05303d7705fa) which makes sense as it is
the deposit smart contract for staked ETHs (17+ million ETH).

#### Example 2: Distinct USDC holders
The following query returns the number of recipient addresses USDC ([0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48](https://etherscan.io/address/0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48)) have been 
transferred to.

```sql
select count(distinct to_address) from `bigquery-public-data.crypto_ethereum.token_transfers`
where token_address = lower('0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'); 
```

This query processes 112.66 GB and took 4 seconds to run.  
The result is 10,668,152 addresses (one holder may own multiple addresses).

## How risky is a given token?

How do I define "risky" - from a financial crime point-of-view?  
Risky here means that out of all the holders of a given token, if we assign a risk score per address, 
how would one derive a risk score for that token?  
(Please note that there are several [categories](https://academy.chainalysis.com/risky-typologies) of risk: sanctions, child abuse, darknet, fraud, ransomware, terrorist financing, ...)  

We first need to get all holders of a token and the notional per holder; we will use GCP for this.  
Next, we derive a risk score per address, using either [TRM Labs](https://www.trmlabs.com/), [Elliptic](https://www.elliptic.co/)
or [Chainalysis](https://www.chainalysis.com/).

Each risk level 'unknown', 'low', 'medium' or 'high' is assigned a numerical value:  unknown = 0, low = 10, medium = 25 and high = 100 
for example.  
We can use a simple weighted average formula to define the "riskiness" of a token: 

$$Risk Score = \sum_{i=1}^n \frac{r_i n_i}{N}$$

### Find all token holders and amounts

This is the query to run on GCP:

```sql
select address, sum(value) as balance from
    (
        select token_address,
               to_address as address,
               cast(value as bignumeric) as value,
        from `bigquery-public-data.crypto_ethereum.token_transfers`

        union all

        select token_address,
            from_address as address,
            -cast(value as bignumeric) as value,
        from `bigquery-public-data.crypto_ethereum.token_transfers`
    )
where token_address = lower('0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48')
group by address
having balance > 0
order by balance desc;
```

This query will process 190.93 GB when run (at this time of writing), took 13 seconds to execute.  
The result is a table with 1,599,449 rows with address and amount.  
From GCP the results can be exported to Google Drive or to a BigQuery table - up to you how you want to process the data.  

Next is to apply the weighted average formula above - that you can adapt according to your risk criteria.

The risk score for this token is therefore 3.59%.  
By risk, this was the allocation of holders: unknown=1,399,484, lowRisk=196,956, highRisk=3,009.

This approach can be used for all ERC20 tokens on any EVM chain.

## How does it relate to DeFi?

When applied to DeFi protocols, such as [Uniswap](https://uniswap.org/), 
using the positions based on the [NFTs minted](https://opensea.io/collection/uniswap-v3-positions), 
it turns out that for a given pool, the pool risk is more or less equivalent to the tokens risks that constitute that pool.

## Conclusion
By utilizing a statistical approach, we can effectively gauge the risk level of a public permission-less DeFi protocol 
or set of tokens over time.  
This method allows us to establish a metric that can be tracked and monitored, providing valuable insights into the 
perceived "riskiness" of a given system.  
Of course, it's important to note that each protocol may require a customized approach, and a thorough examination of other potential hazards, 
including cyber-risks, may also be necessary.  

So, how exactly would one assess DeFi risk?  

A carefully crafted and comprehensive analysis, backed by reliable data and expert insights, is undoubtedly the most effective means 
of evaluating potential risks and vulnerabilities within the DeFi ecosystem.

Thank you to [Yannick Cherel](https://www.linkedin.com/in/yannick-cherel-8b20933/) and [Lucy Janaudy](https://www.linkedin.com/in/lucy-janaudy/)
for their contributions.
