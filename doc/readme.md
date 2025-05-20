# Financial Database Query Templates

This document provides a comprehensive reference for query templates used in the financial database Text2SQL system. Each template maps natural language queries to SQL queries with parameterized values.

## Template Overview

The system includes templates for various financial data queries, ranging from basic company information to complex financial analysis. Templates are structured with:

- **ID**: Unique identifier
- **Natural Language Template**: Human-readable query pattern with parameters
- **Key Parameters**: Required values to complete the query
- **SQL Template**: Corresponding SQL query structure

## Basic Queries

### Company Information

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| basic_company_info | Show me information about {company} | company |

```sql
SELECT companyid, companyname, city, yearfounded, webpage
FROM ciqcompany
WHERE {company_condition}
```

### Geographic Analysis

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| companies_by_country | List companies headquartered in {country} | country |

```sql
SELECT c.companyid, c.companyname, c.city
FROM ciqcompany c
JOIN ciqcountrygeo g ON c.countryid = g.countryid
WHERE {country_condition}
ORDER BY c.companyname
LIMIT 25
```

### Company Profile Queries

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| companies_by_founding_year | Find companies founded {comparison} {year} | comparison, year |

```sql
SELECT companyid, companyname, yearfounded
FROM ciqcompany
WHERE yearfounded {comparison_op} {year_value}
ORDER BY yearfounded DESC, companyname
LIMIT 25
```

## Industry Analysis

### Industry Classification

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| companies_by_industry | Show me {count} companies in the {industry} industry | count, industry |

```sql
SELECT c.companyid, c.companyname, c.city
FROM ciqcompany c
JOIN ciqcompanyindustry ci ON c.companyid = ci.companyid
JOIN industry i ON ci.industryid = i.industryid
WHERE {industry_condition}
ORDER BY c.companyname
LIMIT {limit_value}
```

### Industry Metrics

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| avg_metric_by_industry | What is the average {financial_metric} for companies in {industries}? | financial_metric, industries |

```sql
SELECT i.industryname,
       COUNT(DISTINCT c.companyid) as company_count,
       AVG({metric_field})/1000000 as avg_value_millions
FROM ciqcompany c
JOIN ciqcompanyindustry ci ON c.companyid = ci.companyid
JOIN industry i ON ci.industryid = i.industryid
JOIN ciqmarketcap m ON c.companyid = m.companyid
LEFT JOIN ciqfinancials f ON c.companyid = f.companyid
    AND f.periodtypeid = 1
    AND f.periodenddate = (
        SELECT MAX(periodenddate)
        FROM ciqfinancials
        WHERE companyid = c.companyid
        AND periodtypeid = 1
    )
WHERE i.industryname {industries_condition}
AND m.pricingdate = (SELECT MAX(pricingdate) FROM ciqmarketcap)
GROUP BY i.industryid, i.industryname
ORDER BY avg_value_millions DESC
```

## Financial Metrics

### Market Capitalization

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| companies_by_market_cap | Which companies have a {financial_metric} {comparison} {amount}? | financial_metric, comparison, amount |

```sql
SELECT c.companyid, c.companyname, m.marketcap/1000000000 as market_cap_billions
FROM ciqcompany c
JOIN ciqmarketcap m ON c.companyid = m.companyid
WHERE {metric_condition}
AND m.pricingdate = (SELECT MAX(pricingdate) FROM ciqmarketcap)
ORDER BY m.marketcap DESC
LIMIT 25
```

### Financial Range Queries

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| companies_metric_range | Find {industry} companies with {financial_metric} between {amount1} and {amount2} | industry, financial_metric, amount1, amount2 |

```sql
SELECT c.companyid, c.companyname, {metric_field}/1000000 as value_millions
FROM ciqcompany c
JOIN ciqcompanyindustry ci ON c.companyid = ci.companyid
JOIN industry i ON ci.industryid = i.industryid
JOIN ciqmarketcap m ON c.companyid = m.companyid
LEFT JOIN ciqfinancials f ON c.companyid = f.companyid
    AND f.periodtypeid = 1
    AND f.periodenddate = (
        SELECT MAX(periodenddate)
        FROM ciqfinancials
        WHERE companyid = c.companyid
        AND periodtypeid = 1
    )
WHERE {industry_condition}
AND {metric_field} BETWEEN {amount1_value} AND {amount2_value}
AND m.pricingdate = (SELECT MAX(pricingdate) FROM ciqmarketcap)
ORDER BY {metric_field} DESC
LIMIT 25
```

## Relationship Analysis

### Corporate Structure

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| company_subsidiaries | List all {relationship_type} of {company} | relationship_type, company |

```sql
SELECT c.companyid, c.companyname, c.city, g.country
FROM ciqcompany p
JOIN ciqbusinessrel br ON p.companyid = br.parentcompanyid
JOIN ciqcompany c ON br.childcompanyid = c.companyid
JOIN ciqbusinessreltype brt ON br.businessreltypeid = brt.businessreltypeid
LEFT JOIN ciqcountrygeo g ON c.countryid = g.countryid
WHERE {company_condition}
AND {relationship_condition}
ORDER BY c.companyname
```

## Ranking and Comparisons

### Top Companies

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| top_companies_by_metric | What are the {count} {ranking} companies by {financial_metric}? | count, ranking, financial_metric |

```sql
SELECT c.companyid, c.companyname, {metric_field}/1000000000 as value_billions
FROM ciqcompany c
JOIN ciqmarketcap m ON c.companyid = m.companyid
LEFT JOIN ciqfinancials f ON c.companyid = f.companyid
    AND f.periodtypeid = 1
    AND f.periodenddate = (
        SELECT MAX(periodenddate)
        FROM ciqfinancials
        WHERE companyid = c.companyid
        AND periodtypeid = 1
    )
WHERE m.pricingdate = (SELECT MAX(pricingdate) FROM ciqmarketcap)
ORDER BY {metric_field} DESC
LIMIT {limit_value}
```

### Comparative Analysis

| ID | Natural Language Template | Key Parameters |
|----|--------------------------|----------------|
| compare_companies | Compare {financial_metric} of {companies} | financial_metric, companies |

```sql
SELECT c.companyname, 
       COALESCE({metric_field}/1000000, 0) as value_millions
FROM ciqcompany c
LEFT JOIN ciqmarketcap m ON c.companyid = m.companyid
    AND m.pricingdate = (SELECT MAX(pricingdate) FROM ciqmarketcap)
LEFT JOIN ciqfinancials f ON c.companyid = f.companyid
    AND f.periodtypeid = 1
    AND f.periodenddate = (
        SELECT MAX(periodenddate)
        FROM ciqfinancials
        WHERE companyid = c.companyid
        AND periodtypeid = 1
    )
WHERE c.companyname {companies_condition}
ORDER BY value_millions DESC
```

## Parameter Types

The templates use various parameter types that must be substituted with appropriate values:

| Parameter Type | Description | Examples |
|----------------|-------------|----------|
| company | Company name | Apple, Microsoft, Google |
| country | Country name | United States, Japan, Germany |
| comparison | Comparison operator | greater than, less than, equal to |
| year | Year value | 2020, 2015, 2010 |
| count | Numeric count | 5, 10, 25 |
| industry | Industry category | Technology, Finance, Healthcare |
| financial_metric | Financial measurement | market cap, revenue, P/E ratio |
| amount | Monetary amount | $10 billion, $100 million |
| relationship_type | Business relationship | subsidiary, parent company |
| ranking | Ranking qualifier | largest, highest, smallest |

## Usage Notes

1. Each template requires specific parameter values to generate valid SQL
2. SQL templates include placeholders that must be replaced with contextually appropriate values
3. Some parameters (like company_condition) require special handling to convert natural language to SQL conditions
4. Date-related queries use the most recent data available through subqueries
5. Financial values are typically converted to billions or millions for readability

---

This documentation provides a reference for the query templates used in the financial database Text2SQL system. For implementation details, refer to the parameter substitution framework documentation.