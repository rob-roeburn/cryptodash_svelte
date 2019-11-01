<link rel="stylesheet" href="https://fonts.googleapis.com/icon?family=Material+Icons">
<link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,600,700">
<link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Roboto+Mono">

<script>
// Set dbServer to location of deployed AWS and Lambda solution - Python implementation
const awsLambda = "https://ratossrau3.execute-api.eu-west-1.amazonaws.com/prod"

const dateOptions = { year: 'numeric', month: 'numeric', day: 'numeric', hour: 'numeric', minute: 'numeric', second: 'numeric' }

let tickers = [ { } ];
let positionData = [ { } ];
let selectedTickerId, selectedTickerName, selectedTickerSymbol, selectedTickerPrice = '';
let exchangeRates = [ {'rateName': 'USD', 'rate': 1, 'symbol': '$'} , {'rateName' :'GBP', 'rate': 0.89, 'symbol': '£' } ];
let selectedExchangeRate = {'rateName': 'USD', 'rate': 1, 'symbol': '$' };
let portfolioColumns = [ { title: 'Trade time', field: 'tradetime' },{ title: 'Name', field: 'name' },{ title: 'Symbol', field: 'symbol' },{ title: 'Position', field: 'position' },{ title: 'Price at trade ('+selectedExchangeRate.symbol+')', field: 'tradePrice' },{ title: 'Active', field: 'active' },{ title: 'Unrealised P&L  ('+selectedExchangeRate.symbol+')', field: 'pl' } ];
let portfolioPrecision = 5;
let portfolioId = 0;
let portfolioUnrealisedPL, portfolioRealisedPL = 0;
let clickable = true;
let dtPositions = [];

/**
* Async function to retrieve ticker list and populate to state, and get the price for the first currency to display to the user.
*/
const loadTickers = async e => {
	let tickerList = [];
	const getTickers = await fetch( awsLambda+'/getTickers' );
	const tickerData = await getTickers.json();
	if ( getTickers.status !== 200 ) {
		throw Error( tickerData.message );
	}
	for ( let ticker of JSON.parse( tickerData.body ).Items ) {
		tickerList.push({
			"tickerId": parseInt( ticker.tickerId.N ),
			"tickerName": ticker.tickerName.S,
			"tickerSymbol": ticker.tickerSymbol.S
		});
	}
	const priceresponse = await fetch( awsLambda+'/getPrice/'+tickerList[0].tickerId );
	const pricebody = await priceresponse.json();
	if ( priceresponse.status !== 200 ) {
		throw Error( pricebody.message )
	} else {
		tickers=tickerList;
		selectedTickerPrice=pricebody.Item.cmcCacheData.M.quote.M["USD"].M.price.N;
		selectedTickerId=tickerList[0].tickerId;
		selectedTickerSymbol=tickerList[0].tickerSymbol;
		selectedTickerName=tickerList[0].tickerName;
	}
};

/**
* Async function to retrieve portfolio list and populate to state, and calculate P&L on the fly for live positions.
*/
const refreshPortfolio = async e => {
	const response = await fetch( awsLambda+'/getPortfolio/'+portfolioId );
	const portfolioData = await response.json();
	dtPositions = portfolioData.Item.positions.L;
	if ( response.status !== 200 ) {
		throw Error( portfolioData.message );
	} else {
		let unrealisedPL = 0;
		let portfolioRows = [];
		if ( typeof( portfolioData.Item.positions.L ) !== 'undefined' ) {
			for ( let position of portfolioData.Item.positions.L ) {
				let tickerPrice,positionPL = 0;
				const priceresponse = await fetch( awsLambda+'/getPrice/'+position.M.currencyId.S );
				const pricebody = await priceresponse.json();
				if ( priceresponse.status !== 200 ) {
					throw Error( pricebody.message );
				} else {
					tickerPrice = pricebody.Item.cmcCacheData.M.quote.M["USD"].M.price.N;
					// Only aggregate P&L for active positions
					if( position.M.active.BOOL ) {
						// Calculate P&L - current price - price at trade * position qty
						unrealisedPL = unrealisedPL+( tickerPrice-position.M.priceAtTrade.S )*position.M.positionQty.S;
						positionPL = ( tickerPrice-position.M.priceAtTrade.S )*position.M.positionQty.S;
						// Push each position up to the newData array
						portfolioRows.push({
							id: position.M._id.S,
							portfolioId: portfolioId,
							tradetime: new Date( parseInt( position.M.DateTime.N ) ).toLocaleTimeString( "en-GB", dateOptions ),
							currencyId: position.M.currencyId.S,
							name: position.M.name.S,
							symbol: position.M.symbol.S,
							position: position.M.positionQty.S,
							tradePrice: ( position.M.priceAtTrade.S )*selectedExchangeRate.rate,
							active: position.M.active.BOOL.toString(),
							pl: ( positionPL*selectedExchangeRate.rate ).toFixed( portfolioPrecision )
						})
					}
				}
				portfolioUnrealisedPL=unrealisedPL;
				portfolioRealisedPL=portfolioData.Item.realisedPL.N;
			}
		}
	}
};

/**
* Async function to enter trade data using the selected currency and input position data.
*/
const enterTrade = async e => {
	if ( window.confirm ( "Are you sure?" ) ) {
		let postData = [];
		postData.push( {"portfolioId": portfolioId} );
		postData.push( {"tradeDate": new Date()} );
		postData.push( {"positionQty":document.getElementById("positionQty").value} );
		postData.push( {"tickerId":selectedTickerId} );
		postData.push( {"tickerName":selectedTickerName} );
		postData.push( {"tickerSymbol":selectedTickerSymbol} );
		postData.push( {"tickerPrice":selectedTickerPrice} );
		const response = await fetch( awsLambda+'/postNewPosition', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				'Access-Control-Allow-Methods': '*',
				'Access-Control-Allow-Credentials': 'true',
				'Access-Control-Allow-Headers': 'Content-Type, Authorization'
			},
			body: JSON.stringify({ post: postData }),
		});
		const body = await response;
		if ( response.status !== 200 ) {
			throw Error(body.message)
		} else {
			refreshPortfolio();
		}
	}
};

/**
* Async function to reset the cache file in the database based on the button value.
*/
const updateCacheFile = async e => {
	const response = await fetch(awsLambda+'/updateCache?file='+e.target.textContent);
	const body = await response;
	if (response.status !== 200) {
		throw Error(body.message);
	} else {
		loadTickers();
		refreshPortfolio();
	}
};

/**
* Async function to reset the portfolio view in the database completely.
*/
const resetPortfolio = async e => {
	if ( window.confirm( "Are you sure?" ) ) {
		const response = await fetch( awsLambda+'/resetPortfolio', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
			},
			body: JSON.stringify( { post: [ { "portfolioId": portfolioId} ] } ),
		});
		const body = await response;
		if (response.status !== 200) {
			throw Error(body.message);
		} else {
			setTimeout(() => {
				refreshPortfolio();
			}, 250);
		}
	}
};

/**
* Async function to retrieve price data for a ticker defined by the CMC ID.
*/
const getPrice = async e => {
	let tickerData = searchObject( tickers, e.target.value,"tickerId" )
	const priceresponse = await fetch( awsLambda+'/getPrice/'+parseInt( e.target.value ) )
	const pricebody = await priceresponse.json()
	if ( priceresponse.status !== 200 ) {
		throw Error( pricebody.message )
	} else {
		selectedTickerId=e.target.value;
		selectedTickerPrice = pricebody.Item.cmcCacheData.M.quote.M["USD"].M.price.N.toString()
		selectedTickerSymbol=tickerData[0].tickerSymbol;
		selectedTickerName=tickerData[0].tickerName;
	}
};

/**
* Helper function to search object and match on key names
*/
const searchObject = (obj, match, field) => {
	let results = [];
	for( let i = 0; i<obj.length; i++ ) {
		if( obj[i][field] === parseInt( match ) ) {
			results.push( obj[i] );
		}
	}
	return results;
};

/**
* Helper function to update exchange rate state based on dropdown selection
*/
const updateExchangeRate = (e) =>  {
	selectedExchangeRate = exchangeRates.find( exchangeRateData => exchangeRateData.rateName === e.target.value );
	portfolioColumns[4].title="Price at trade ("+selectedExchangeRate.symbol+")";
	portfolioColumns[6].title="Unrealised P&L ("+selectedExchangeRate.symbol+")";
};

loadTickers();
refreshPortfolio();

</script>


<table ref="table" class="table table-striped table-hover" style="width:100%">
<tbody>
<tr>
<td>
<h1>CryptoDash</h1>
</td>
<td>
<h4>Unrealised P&L Total: { selectedExchangeRate.symbol+''+( portfolioUnrealisedPL * selectedExchangeRate.rate ).toFixed( portfolioPrecision ) }</h4>
</td>
<td>
<h4>Realised P&L Total: { selectedExchangeRate.symbol+''+( portfolioRealisedPL * selectedExchangeRate.rate ).toFixed( portfolioPrecision ) }</h4>
</td>
</tr>
</tbody>
</table>


<hr><h3>Portfolio View</h3>

<table ref="table" class="table table-striped table-hover" style="width:100%">
<thead>
<tr>
{#each portfolioColumns as column}
<th style="width: { column.width ? column.width : 'auto' }" align="center">
{column.title}
</th>
{/each}
</tr>
</thead>
<tbody>
{#each dtPositions as row}
<tr>
<td align="center">{new Date(parseInt(row.M.DateTime.N)).toLocaleTimeString("en-GB" , dateOptions )}</td>
<td align="center">{row.M.name.S}</td>
<td align="center">{row.M.symbol.S}</td>
<td align="center">{row.M.positionQty.S}</td>
<td align="center">{(parseFloat(row.M.priceAtTrade.S)*selectedExchangeRate.rate).toFixed(portfolioPrecision)}</td>
<td align="center">{row.M.active.BOOL}</td>
<td align="center">{row.M.PL.N}</td>
</tr>
{/each}
</tbody>
</table>
<br>


<hr><h3>Trade Entry</h3>

<table ref="table" class="table table-striped table-hover" style="width:50%">
<thead>
<tr>
<th style="width: 'auto' }" align="center">
Select currency
</th>
<th style="width: 'auto' }" align="center">
Price of currency
</th>
<th style="width: 'auto' }" align="center">
Position quantity
</th>
<th>
</th>
</tr>
</thead>
<tbody>
<tr>
<td align="center">
<select on:change="{getPrice}">{#each tickers as ticker}<option value={ticker.tickerId}>{ticker.tickerName}</option>{/each}</select>
</td>
<td align="center">
{ selectedExchangeRate.symbol +''+ ( selectedTickerPrice * selectedExchangeRate.rate ) }
</td>
<td align="center">
<input id='positionQty' type='text' />
</td>
<td align="center">
<button on:click={enterTrade}>Trade</button>
</td>
</tr>
</tbody>
</table>


<hr><h3>System Control</h3>

<table ref="table" class="table table-striped table-hover" style="width:50%">
<thead>
<tr>
<th style="width: 'auto' }" align="center">
Reset portfolio
</th>
<th style="width: 'auto' }" align="center">
Currency to view ( USD={ selectedExchangeRate.rate } )
</th>
</tr>
</thead>
<tbody>
<tr>
<td align="center">
<button on:click={resetPortfolio}>Zero positions</button>
</td>
<td align="center">
<select on:change={updateExchangeRate}>{#each exchangeRates as rate}<option value={rate.rateName}>{rate.rateName}</option>{/each}</select>
</td>
</tr>
</tbody>
</table>
