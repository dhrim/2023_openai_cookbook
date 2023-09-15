
# 코드 분석

```
nl_query: str = "Return the name of each department that had more than 10 employees in June 2021"
eval_template: str = "{};\n-- Explanation of the above query in human readable format\n-- {}"

table_definitions: str = """
# Employee(id, name, department_id)
# Department(id, name, address)
# Salary_Payments(id, employee_id, amount, date)
"""

prompt_template: str = """
### Postgres SQL tables, with their properties:
#{}#
### {}
{}
"""

result = generate_sql(
    prompt_template,
    table_definitions,
    nl_query,
    eval_template,
)
print(result)
```

<br>

## def generate_sql()
```
def generate_sql(
    prompt_template: str,
    table_definitions: str,
    nl_query: str,
    eval_template: str,
    return_all_results: bool = False,
) :

    # prompt를 만들고
    prompt = prompt_template.format(
        table_definitions, nl_query, SQL_STARTS
    )

    # 호출하여 SQL을 생성한다.
    responses = get_candidates(prompt)

    # 평가한다.
    candidates = []
    for response in responses:
        quality = eval_candidate(response, nl_query, eval_template)
        candidates.append((response, quality))

    # 소팅하고
    candidates.sort(key=lambda x: x[1], reverse=True)

    if return_all_results:
        return candidates

    # 최고 점수의 것 반환        
    return candidates[0][0]

```

SQL을 생성하기 위하여 OpenAI API를 호출할 때 사용되는 prompt는 다음과 같다.
```
### Postgres SQL tables, with their properties:
#
# Employee(id, name, department_id)
# Department(id, name, address)
# Salary_Payments(id, employee_id, amount, date)
#
### Return the name of each department that had more than 10 employees in June 2021
SELECT
```

<br>

# def get_candidates()

```
def get_candidates(
    prompt: str,
) -> List[str]:
    response = openai.Completion.create(
        prompt=prompt,
        engine=ENGINE,
        temperature=TEMPERATURE,
        max_tokens=150,
        top_p=1,
        frequency_penalty=0,
        presence_penalty=0,
        stop=GENERATING_STOPS,
        n=N,
    )

    responses = [SQL_STARTS + choice.text for choice in response.choices]
    return responses

```

OpenAI 호출을 통해 받은 response는 다음과 같다.
```
{
  "warning": "This model version is deprecated. Migrate before January 4, 2024 to avoid disruption of service. Learn more https://platform.openai.com/docs/deprecations",
  "id": "cmpl-7yEQgYgc1K9NLXCIqAr4kScyyaZgQ",
  "object": "text_completion",
  "created": 1694589502,
  "model": "text-davinci-003",
  "choices": [
    {
      "text": "    d.name\nFROM\n    Department d\nINNER JOIN\n    Employee e\nON\n    d.id = e.department_id\nWHERE\n    EXTRACT(MONTH FROM date) = 6\n    AND EXTRACT(YEAR FROM date) = 2021\nGROUP BY\n    d.name\nHAVING\n    COUNT(e.id) > 10",
      "index": 0,
      "logprobs": null,
      "finish_reason": "stop"
    },
    {
      "text": "    d.name\nFROM\n    Department d\n    INNER JOIN Employee e ON d.id = e.department_id\n    INNER JOIN Salary_Payments sp ON e.id = sp.employee_id\nWHERE\n    sp.date BETWEEN '2021-06-01' AND '2021-06-30'\nGROUP BY\n    d.name\nHAVING\n    COUNT(e.id) > 10",
      "index": 1,
      "logprobs": null,
      "finish_reason": "stop"
    },
    {
      "text": "    d.name\nFROM\n    Department d\nINNER JOIN\n    Employee e\nON\n    d.id = e.department_id\nINNER JOIN\n    Salary_Payments sp\nON\n    e.id = sp.employee_id\nWHERE\n    sp.date BETWEEN '2021-06-01' AND '2021-06-30'\nGROUP BY\n    d.name\nHAVING\n    COUNT(e.id) > 10",
      "index": 2,
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 74,
    "completion_tokens": 298,
    "total_tokens": 372
  }
}
```

SQL이 N개 생성되고


<br>

# def eval_candidate()

```
def eval_candidate(
    generated_sql: str,
    nl_query: str,
    eval_template: str,
    answer_start_token: str = "--",
) -> float:

    prompt = eval_template.format(generated_sql, nl_query)

    response = openai.Completion.create(
        engine=ENGINE,
        prompt=prompt,
        temperature=0,
        max_tokens=0,
        top_p=1,
        frequency_penalty=0,
        presence_penalty=0,
        logprobs=1,
        echo=True,
    )

    answer_start = rindex(
        response["choices"][0]["logprobs"]["tokens"], answer_start_token
    )
    logprobs = response["choices"][0]["logprobs"]["token_logprobs"][answer_start + 1 :]
    return sum(logprobs) / len(logprobs)

```


생성된 SQL에 대한 사람이 읽을 수 있는 포맷으로 설명하라는 prompt는 아래와 같다.


```
SELECT    Department.name
FROM
    Employee
INNER JOIN
    Department ON Employee.department_id = Department.id
INNER JOIN
    Salary_Payments ON Employee.id = Salary_Payments.employee_id
WHERE
    Salary_Payments.date BETWEEN '2021-06-01' AND '2021-06-30'
GROUP BY
    Department.name
HAVING
    COUNT(Employee.id) > 10;
-- Explanation of the above query in human readable format
-- Return the name of each department that had more than 10 employees in June 2021
```

이에 대한 response는 다음과 같다.

```
{
  "id": "cmpl-7yEU2yHJvlmb95oG3mf4Cz5hfOFdl",
  "object": "text_completion",
  "created": 1694589710,
  "model": "text-davinci-003",
  "choices": [
    {
      "text": "SELECT    d.name\nFROM\n    Department d\n   ...",
      "index": 0,
      "logprobs": {
        "tokens": [
          "SELECT",
          "   ",
          " d",
          ".",
          "name",
          "\n",
          "FR",
          "OM",
          "\n",
          "   ",

          ...

        ],
        "token_logprobs": [
          null,
          -6.8370504,
          -4.4920807,
          -1.5437704,
          -2.6638978,
          -1.6289743,

          ...

        ],
        "top_logprobs": [
          null,
          {
            " *": -1.150973
          },
          {
            "\n": -0.97546
          },
          {
            "bo": -0.3934572
          },
          {
            "id": -2.4390433
          },

          ...

        ],
        "text_offset": [
          0,
          6,
          9,
          11,
          12,
          16,

          ...

       ]
      },
      "finish_reason": "length"
    }
  ],
  "usage": {
    "prompt_tokens": 142,
    "total_tokens": 142
  }
}
```

choices > logprobs > token_logprobs의 값을 가지고 이값들이 크면 좋은 SQL이라는데... 

logprobs에 대한 설명 : http://gptprompts.wikidot.com/intro:logprobs

또다른 설명 : https://blog.scottlogic.com/2021/09/01/a-primer-on-the-openai-api-2.html








