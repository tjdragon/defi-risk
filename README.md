# DeFi Risk

What is the best approach to DeFi risk?  
March, 15th 2023

## Intro

Decentralised Finance (DeFi) is about enabling digital assets based transactions, 
exchanges and financial services  
DeFi's core premise is that there is no centralized authority to dictate or control operations.  

But how to assess DeFi risk, specifically risk related to cyber-crime?

## Token risk

We will first start by looking at token risk. For the rest of this article, tokens relate
to ERC20 tokens on the Ethereum ecosystem, but the same principal would apply elsewhere.  

### An ERC20 primer
The [ERC20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) standard does not allow to list
all its participants, meaning it is possible to say how much tokens an address has, but it is not possible
using the standard ERC20 functions to list all the holders of that token.  

For this, we need to rely on the 'transfer' events. For this we need to use an archive node to be able to 
get all the events in and out from an ERC20.  

### Querying the data
There are a couple of ways to do this, the DIY version, which I do not recommend, the use of [TheGraph](https://thegraph.com/) which
is not the faint-hearted, and the one I have used, the lazy but very powerful and efficient option: [GCP](https://cloud.google.com/).  

Google has made several blockchain databases available that can be queried using [BigQuery](https://cloud.google.com/bigquery).  

#### Example 1: Top 10 ETH addresses by balance
Follows an example to get the top 10 addresses by balance on Ethereum:

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
Follows an example to returns the number of 'TO' addresses USDC ([0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48](https://etherscan.io/address/0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48)) has been 
transferred to.

```sql
select count(distinct to_address) from `bigquery-public-data.crypto_ethereum.token_transfers`
where token_address = lower('0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'); 
```

This query processes 112.66 GB and took 4 seconds to run.  
The result is 10,668,152 addresses.

## How risky is USDC?

How do I define "risky"? Risky here means that out of all the holders of USDC, if we assign a risk score per address,
how would derive a risk score for the token?  

In order to do so, we first need to get all holders of a token and the notional per holder. We will use GCP for this.  
Next, we need to derive a risk score per address. Using either [TRM Labs](https://www.trmlabs.com/), [Elliptic](https://www.elliptic.co/) 
or [Chainalysis](https://www.chainalysis.com/), you can get a risk score in the form of 'unknown', 'low', 'medium' or 'high'.  
Next is to assign a numerical risk score for each value: unknown = 0, low = 10, medium = 25 and high = 100 for example.  
We can use a simple weighted average formula to define the "riskiness" of a token: 

$$Risk Score = \sum_{i=1}^n \frac{r_i n_i}{N}$$

### Find all USDC holders and amounts

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

The risk score for USDC is therefore 3.59%.  
By risk, this was the allocation of holders: unknown=1,399,484, lowRisk=196,956, highRisk=3,009.

This approach can be used for all ERC20 tokens on any EVM chain.

## How does it relate to DeFi?

I started to look at applying this to DeFi protocols, such as [Uniswap](https://uniswap.org/), 
using the positions based on the [NFTs minted](https://opensea.io/collection/uniswap-v3-positions).  
It turns out that for a given pool, the pool risk is more or less equivalent to the tokens risks that constitute that pool.  

## Conclusion
I believe that only a statistical approach to public permission-less DeFi is possible. What's your view?
