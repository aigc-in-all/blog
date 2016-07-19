title: Java四则运算
date: 2016-07-16 22:42:10
tags: Arithmetic
---

1. 遇到数字则直接压到数字栈顶
2. 遇到运算符`（+-*/）`时，若操作符栈为空，则直接放到操作符栈顶，否则，见**3**
3. 若操作符栈顶元素的优先级比当前运算符的优先级小，则直接压入栈顶，否则执行步骤**4**
4. 弹出数字栈顶的两个数字并弹出操作符栈顶的运算符进行运算，把运算结果压入数字栈顶，重复**2**和**3**直到当前运算符被压入操作符栈顶
5. 遇到左括号`(`时则直接压入操作符栈顶。
6. 到右括号`)`时则依次弹出操作符栈顶的运算符运算数字栈的最顶上两个数字，直到弹出的操作符为左括号

<!-- more -->

```java
public class Demo {

	public static void main(String[] args) {
		String str = "12 + 3 * (2 + 5) - 44 / (12 - 8) + 4 * ( 12 + 2 * (4 + 16) + 3)";
		double result = new Demo().computeWithStack(str);
		System.out.println(result);
	}

	public double computeWithStack(String computeExpr) {
		StringTokenizer tokenizer = new StringTokenizer(computeExpr, "+-*/()", true);
		Stack<Double> numStack = new Stack<Double>();
		Stack<Operator> operatorStack = new Stack<Operator>();
		Map<String, Operator> operatorMap = getOperatorMap();
		String current;
		while (tokenizer.hasMoreTokens()) {
			current = tokenizer.nextToken().trim();
			if (current == null || current.length() == 0) {
				continue;
			}

			if (this.isNum(current)) {
				numStack.push(Double.valueOf(current));
				continue;
			}

			Operator operator = operatorMap.get(current);
			if (operator != null) {
				while (!operatorStack.empty() && operatorStack.peek().priority() >= operator.priority()) {
					compute(numStack, operatorStack);
				}

				operatorStack.push(operator);
			} else {
				if ("(".equals(current)) {
					operatorStack.push(Operator.BRACKETS);
				} else {
					while (!operatorStack.peek().equals(Operator.BRACKETS)) {
						compute(numStack, operatorStack);
					}
					operatorStack.pop();
				}
			}
		}

		while (!operatorStack.empty()) {
			compute(numStack, operatorStack);
		}

		return numStack.pop();
	}

	private boolean isNum(String str) {
		String numRegex = "^\\d+(\\.\\d+)?$";
		return Pattern.matches(numRegex, str);
	}

	private Map<String, Operator> getOperatorMap() {
		Map<String, Operator> map = new HashMap<>();
		map.put("+", Operator.PLUS);
		map.put("-", Operator.MINUS);
		map.put("*", Operator.MULTIPLY);
		map.put("/", Operator.DIVIDE);
		return map;
	}

	private void compute(Stack<Double> numStack, Stack<Operator> operStack) {
		Double num2 = numStack.pop();
		Double num1 = numStack.pop();
		Double result = operStack.pop().compute(num1, num2);
		numStack.push(result);
	}
}
```

```java
public enum Operator {

	PLUS {
		@Override
		public int priority() {
			return 1;
		}

		@Override
		public double compute(double num1, double num2) {
			return num1 + num2;
		}
	},

	MINUS {
		@Override
		public int priority() {
			return 1;
		}

		@Override
		public double compute(double num1, double num2) {
			return num1 - num2;
		}
	},

	MULTIPLY {
		@Override
		public int priority() {
			return 2;
		}

		@Override
		public double compute(double num1, double num2) {
			return num1 * num2;
		}
	},

	DIVIDE {
		@Override
		public int priority() {
			return 2;
		}

		@Override
		public double compute(double num1, double num2) {
			return num1 / num2;
		}
	},

	BRACKETS {
		@Override
		public int priority() {
			return 0;
		}

		@Override
		public double compute(double num1, double num2) {
			return 0;
		}
	};

	public abstract int priority();

	public abstract double compute(double num1, double num2);
}
```