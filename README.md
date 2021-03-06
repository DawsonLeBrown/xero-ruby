# xero-ruby
Xero Ruby SDK for OAuth 2.0 generated from [Xero API OpenAPI Spec](https://github.com/XeroAPI/Xero-OpenAPI).

[![RubyGem](https://img.shields.io/badge/xero--ruby%20gem-v0.2.4-brightgreen)](https://rubygems.org/gems/xero-ruby)

# Documentation
Xero Ruby SDK supports Xero's OAuth2.0 authentication and supports the following Xero API sets.

## APIS
* [Accounting Api Docs](/docs/accounting/AccountingApi.md)
* [Asset Api Docs](/docs/assets/AssetApi.md)
* [Project Api Docs](docs/projects/ProjectApi.md)

## Models
* [Accounting Models Docs](/docs/accounting/)
* [Asset Models Docs](/docs/assets/)
* [Project Models Docs](/docs/projects/)

### Coming soon
* payroll (AU)
* payroll (NZ/UK)
* files
* xero hq
* bank feeds

## Sample Apps
We have two apps showing SDK usage.
* https://github.com/XeroAPI/xero-ruby-oauth2-starter (**Sinatra** - session based / getting started)
* https://github.com/XeroAPI/xero-ruby-oauth2-app (**Rails** - token management / full examples)

## Looking for OAuth 1.0a support?
Check out the [Xeroizer](https://github.com/waynerobinson/xeroizer) gem (maintained by community).

---
## Installation
To install this gem to your current gemset.
```
gem install 'xero-ruby'
```
Or add to your gemfile and run `bundle install`.
```
gem 'xero-ruby'
```

## Getting Started
* Create a [free Xero user account](https://www.xero.com/us/signup/api/)
* Login to your Xero developer [/myapps](https://developer.xero.com/myapps) dashboard & create an API application and note your API app's credentials.

### Creating a Client
* Get the credential values from an API application at https://developer.xero.com/myapps/.
* Include [neccesary scopes](https://developer.xero.com/documentation/oauth2/scopes) as comma seperated list
    * example => "`openid profile email accounting.transactions accounting.settings`"
```
require 'xero-ruby'
```
```ruby
creds = {
  client_id: ENV['CLIENT_ID'],
  client_secret: ENV['CLIENT_SECRET'],
  redirect_uri: ENV['REDIRECT_URI'],
  scopes: ENV['SCOPES']
}
xero_client ||= XeroRuby::ApiClient.new(credentials: creds)
```

## User Authorization & Callback
All API requests require a valid access token to be set on the client.

To generate a valid `token_set` send a user to the `authorization_url`:
```ruby
@authorization_url = xero_client.authorization_url

redirect_to @authorization_url
```

Xero will then redirect back to the URI defined in your ENV['REDIRECT_URI'] variable.
*This must match **exactly** with the variable in your /myapps dashboard.*

In your callback route catch, calling `get_token_set_from_callback` will exchange the temp code in your params, with a valid `token_set` that you can use to make API calls.
```ruby
# => http://localhost:3000/oauth/callback

token_set = xero_client.get_token_set_from_callback(params)

# save token_set JSON in a datastore in relation to the user authentication
```

## Making API calls once you have a token_set
For use outside of the initial auth flow, setup the client by passing the whole token_set to `refresh_token_set` or `set_token_set`.
```ruby
xero_client.refresh_token_set(user.token_set)

xero_client.set_token_set(user.token_set)
```
A `token_set` contains data about your API connection most importantly :
* `access_token`
* `refresh_token`
* `expiry`

**An `access_token` is valid 30 minutes and a `refresh_token` is valid for 60 days**

Example Token set:
> You can decode the `id_token` & `access_token` for additional metadata by using a [decoding library](https://github.com/jwt/ruby-jwt):
```json
{
  "id_token": "xxx.yyy.zz",
  "access_token": "xxx.yyy.zzz",
  "expires_in": 1800,
  "token_type": "Bearer",
  "refresh_token": "xxxxxx",
  "scope": "email profile openid accounting.transactions offline_access"
}
```

## Token & SDK Helpers
Refresh/connection helpers
```ruby
@token_set = xero_client.refresh_token_set(user.token_set)

# Xero's tokens can potentially facilitate (n) org connections in a single token. It is important to store the `tenantId` of the Organisation your user wants to read/write data.

# The `updatedDateUtc` will show you the most recently authorized Tenant (AKA Organisation)
connections = xero_client.connections
[{
  "id" => "xxx-yyy-zzz",
  "tenantId" => "xxx-yyy-zzz",
  "tenantType" => "ORGANISATION",
  "tenantName" => "Demo Company (US)",
  "createdDateUtc" => "2019-11-01T20:08:03.0766400",
  "updatedDateUtc" => "2020-04-15T22:37:10.4943410"
}]

# disconnect an org from a user's connections. Pass the connection ['id'] not ['tenantId']. Useful if you want to enforce only a single org connection per token.
remaining_connections = xero_client.disconnect(connections[0]['id'])

# set a refreshed token_set
token_set = xero_client.set_token_set(user.token_set)

# access token_set once it is set on the client
token_set = xero_client.token_set
```

Example token expiry helper
```ruby
require 'jwt'

def token_expired?
  token_expiry = Time.at(decoded_access_token['exp'])
  token_expiry > Time.now
end

def decoded_access_token
  JWT.decode(token_set['access_token'], nil, false)[0]
end
```

## API Usage
```ruby
  require 'xero-ruby'

  xero_client.refresh_token_set(user.token_set)

  tenant_id = user.active_tenant_id
  # example of how to store the `tenantId` of the specific tenant (aka organisation)

  # https://github.com/XeroAPI/xero-ruby/blob/master/accounting/lib/xero-ruby/api/accounting_api.rb
  
  # Get Accounts
  accounts = xero_client.accounting_api.get_accounts(tenant_id).accounts

  # Create Invoice
  invoices = { invoices: [{ type: XeroRuby::Accounting::Invoice::ACCREC, contact: { contact_id: contacts[0].contact_id }, line_items: [{ description: "Big Agency", quantity: BigDecimal("2.0"), unit_amount: BigDecimal("50.99"), account_code: "600", tax_type: XeroRuby::Accounting::TaxType::NONE }], date: "2019-03-11", due_date: "2018-12-10", reference: "Website Design", status: XeroRuby::Accounting::Invoice::DRAFT }]}
  invoice = xero_client.accounting_api.create_invoices(tenant_id, invoices).invoices.first

  # Create History
  payment = xero_client.accounting_api.get_payments(tenant_id).payments.first
  history_records = { history_records: [{ details: "This payment now has some History!" }]}
  payment_history = xero_client.accounting_api.create_payment_history(tenant_id, payment.payment_id, history_records)

  # Create Attachment
  account = xero_client.accounting_api.get_accounts(tenant_id).accounts.first
  file_name = "an-account-filename.png"
  opts = {
    include_online: true
  }
  file = File.read(Rails.root.join('app/assets/images/xero-api.png'))
  attachment = xero_client.accounting_api.create_account_attachment_by_file_name(tenant_id, @account.account_id, file_name, file, opts)

  # https://github.com/XeroAPI/xero-ruby/blob/master/accounting/lib/xero-ruby/api/asset_api.rb
  
  # Create Asset
  asset = {
    "assetName": "AssetName: #{rand(10000)}",
    "assetNumber": "Asset: #{rand(10000)}",
    "assetStatus": "DRAFT"
  }
  asset = xero_client.asset_api.create_asset(tenant_id, asset)

  # https://github.com/XeroAPI/xero-ruby/blob/master/docs/projects/ProjectApi.md

  # Get Projects
  projects = xero_client.project_api.get_projects(tenant_id).items
```

## BigDecimal
All monetary and fields and a couple quantity fields utilize BigDecimal
```ruby
  puts invoice.unit_amount
  => 0.2099e2
  
  puts invoice.unit_amount.class 
  => BigDecimal

  puts invoice.unit_amount.to_s("F")
  => "20.99"

  # Rails method-number_to_currency
  number_to_currency(invoice.unit_amount, :unit => "$")
```

## Querying & Filtering
Examples for the `opts` (_options_) parameters most endpoints support.
```ruby
# Invoices
opts = { 
  statuses: [XeroRuby::Accounting::Invoice::PAID],
  where: { amount_due: '=0' },
  if_modified_since: (DateTime.now - 1.hour).to_s
}
xero_client.accounting_api.get_invoices(tenant_id, opts).invoices

# Contacts 
opts = {
  if_modified_since: (DateTime.now - 1.weeks).to_s,
  order: 'UpdatedDateUtc DESC',
  where: {
    is_customer: '==true',
    is_supplier: '==true',
  }
}
xero_client.accounting_api.get_contacts(tenant_id, opts).contacts

# Bank Transactions
opts = {
  if_modified_since: (DateTime.now - 1.year).to_s,
  where: { type: %{=="#{XeroRuby::Accounting::BankTransaction::SPEND}"}},
  order: 'UpdatedDateUtc DESC',
  page: 2,
  unitdp: 4 # (Unit Decimal Places)
}
xero_client.accounting_api.get_bank_transactions(tenant_id, opts).bank_transactions

# Bank Transfers
opts = {
  where: {
    amount: "> 999.99"
  },
  order: 'Amount ASC'
}
xero_client.accounting_api.get_bank_transfers(tenant_id, opts).bank_transfers
```
### NOTE
1) Not all `opts` parameter combinations are available for all endpoints, and there are likely some undiscovered edge cases. If you encounter a filter / sort / where clause that seems buggy open an issue and we will dig.

2) Some opts string values may need PascalCasing to match casing defined in our [core API docs](https://developer.xero.com/documentation/api/api-overview).
    * `opts = { order: 'UpdatedDateUtc DESC'}`

3) If you have use cases outside of these examples let us know.

## Sample App
The best resource to understanding how to best leverage this SDK is the sample applications showing all the features of the gem.
> https://github.com/XeroAPI/xero-ruby-oauth2-starter (Sinatra - simple getting started)
> https://github.com/XeroAPI/xero-ruby-oauth2-app (Rails - full featured examples)