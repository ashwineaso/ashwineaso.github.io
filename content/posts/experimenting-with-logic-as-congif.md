+++ 
draft = false
date = 2021-01-05
title = "Experimenting with business logic as configuration"
description = ""
slug = "experimenting-with-logic-as-config"
authors = ["Ashwin Easo Zachariah"]
tags = []
categories = []
externalLink = ""
series = []
+++

---

## A little byte of context
At Credy, a microfinance company, we provide small-ticket loans to individuals. The ticket size ranges from 10K to 100K INR, over a period of 6–18 months. We evaluate our customers, just like any other bank, based on several factors such as their income, monthly expense, existing loans, credit history, spending patterns etc.

Once the required data is collected, an automated evaluation is run against the loan application, and the application is either approved or rejected. If the evaluator is unable to reach a decision 🤖 (eg: customer seems to have an unusual pattern of transactions in their bank statement. Compulsive gambling maybe? 🏇🏼), it assigns the case for manual evaluation.
So how does the "evaluation" happen? Well, people might (mis)lead you to believe that it's ML and AI, but it's not. It's just hundreds and thousands of rules, against which the application is checked, and then a conclusion is made based upon it.


## No one rule, to rule them all
As our portfolio grew, we started to have different sets of rules for each customer based on where they lived, or what they did, or even how much they spent. Initially, we tried writing the business logic/ rule/ decision tree within the codebase, painstakingly defining the conditions for each case. But that meant that we had to roll out a change for every minor update in the policy. Also, as the rules were rapidly changing, we didn't have a way to keep track of the policy and parameters a specific loan application was evaluated against.

With the goal of moving the business logic outside the codebase, we experimented with several methods, tools and libraries. While some helped model business rules and processes, others were more suited towards running experiments and versioning changes. However, we wanted a generic solution that could give us fine-grained control over all our conditional logic.


## But, does "logic" make sense?
The first step was to decide how the policy would be evaluated. A typical rule would be of the format:

> Approve the loan application if the customer is employed, and has a monthly income of more than 20K INR or has a credit score greater than 800.

The above set of conditions, replaced with the corresponding operators could be written as:

```
is_employed EQ True AND (monthly_income GTE 20000 OR credit_score GTE 800)
```

To make the parsing a lot easier for us, we decided to define the grammar in a way that the expressions were written using the prefix notation (inspired by Lisp). The expression was then restructured to

```
AND (EQ is_employed True) (OR (GTE monthly_income 20000) (GTE credit_score 800))
```

After taking an inventory of the operators we would need, we classified them into Unary, Binary and Nary operators and then defined the business logic for handling the operands. 

Consider the example of a simple "greater than or equal to" operator (GTE):

```python
class GTEOperator(BinaryOperator):

    def get_binary_result(self, operand_1, operand_2):
        """Get result for the given binary operator."""
        return operand_1 >= operand_2
```

In the base class of the operator, we defined the logic common to all such operators, including validation.

```python
class BinaryOperator(JSONOperator):
    """Base class to represent binary operators."""
    def get_result(self):
        """Get result for the given binary operator."""
        # Business logic to check the operands 
        # and evaluate the result of the operation

    def check_valid_json_operands(self, operands):
        """Check if json operands are valid or not."""
        # If there are more that 2 operands for a binary operator
        # then the expression cannot be evaluated
        if len(operands) != 2:
            error_string = '{} should have exactly 2 operands'.format(
                self.__class__.__name__)
            raise ValueError(error_string)
        return True
     
     ...
```

Now the tricky part was to evaluate the operands which are "variables". Take the following example

```
AND (EQ is_employed True) (OR (GTE monthly_income 20000) (GTE credit_score 800))
```

Here, is_employed , monthly_income and credit_score are the variables which have to be replaced with their actual values. For this, we built an interpreter into which we passed the mapping of the variable name to their actual values.

```json
{
    "is_employed": True,
    "monthly_income": 30000,
    "credit_score": 823
}
```

The interpreter replaces the variable names with the values, and evaluates the conditional expression 💪 , returning a True or False . This way, we were able to evaluate hundreds of conditions in a single expression without having to write code for any of them. We finally had a domain-specific language of our own!

## Where did the rules go?

Moving the rules out of the code meant that we were free to store them wherever we wanted, and change them whenever we wanted, with minimal to no effort. To fit our use case, we decided to store them as JSON in our primary database: Postgres. And since Postgres supported JSON Datatype, it seemed like the obvious choice.

We stored the rules in a table along with a name, category and version

| id | name                 | category    | rule                                                                                                        | version |
|----|----------------------|-------------|-------------------------------------------------------------------------------------------------------------|---------|
| 1  | eligibility_policy_1 | eligibility | ["AND", ["EQ", is_employed", true], ["OR, ["GTE", "monthly_income", 20000], ["GTE", "credit_score", 800]]]  | latest  |
|    |                      |             |                                                                                                             |         |


A rule stored in a table.

For each product in the `loan_product` table, we linked the product with its corresponding rules in the `rules`` table.

Products stored in a table with links to their corresponding rules.

| id | name          | eligibility_rule | approval_rule | loan_terms_rule | version |
|---:|---------------|------------------|---------------|-----------------|---------|
| 1  | personal_loan | 1                | 3             | 11              | latest  |
| 2  | auto_loan     | 10               | 12            | 23              | latest  |
|    |               |                  |               |                 |         |

In this way, we could fetch the corresponding rule from the rules table, for each type of rule in the `loan_product` table, and run the evaluation against it.

We added a couple of bells and whistles to the framework too:

1. Versioning: We needed to maintain every version of policy change that had ever happened so that we could later determine what the result of the evaluation of a loan application against a previous version would have been. Or even simply understand how our policies had evolved.

2. Complex Operators: As our use cases expanded, we added more operators into it such as range, map, filter and reduce; and even a couple of complicated ones that were specific to our domain such as the PMT operator: to calculate the payment for a loan based on constant payments and a constant interest rate.
 
3. Caching: As the volume of the cases we had to evaluate increased, we ensured that the database queries to fetch the policy was limited to the bare minimum by caching the policies that were frequently being used.

The success of the framework inspired us to use the same in other parts of the system to answer questions like
- What kind of notifications do we send the customer to remind them about the upcoming EMI payment, and when should we send them? 
- Given the information we have about the users, what should be the price offered to the user to purchase the product?
