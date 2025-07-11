
---
title: Dumb git diff (Part 1)
draft: false
tags:
  - cpp
---
Implementing git diff for yourself is kind of an interesting problem. The goal is to distinguish the differences between two files. Maybe instead finding the differences, let's find the commonalities.

If I were to provide you with 2 strings, how would you calculate the longest common subsequence for the given strings? This is just a dynamic programming problem...

```c++
int lcs(string const& left, string const& right)
{
	// We need to identify the iterative/recursive step
	// There's 2 strings so its probably dp(i, j)
	// left = 'a', right = 'a'
	// dp(0, 0) = int(left[0] == right)

	// left = 'ab', right = 'abc'
	// dp(0,0) = 1
	// dp(0,1) = 1
	// dp(1,0) = 1
	// dp(1,1) = 2 = max( left[1]==right[1] + dp(0,0), dp(0,1), dp(1,0) )
	// dp(1,2) = max( left[1]==right[2] + dp(0,1), dp(0,2), dp(1,1) )

	// Hence, dp(i,j) = max( left[i]==right[j] + dp(i-1, j-1), dp(i-1, j), dp(i, j-1) )

	int r = left.size();
	int c = right.size();
	if(r == 0 || c == 0) return 0;
	vector<vector<int>> lcs_table(r, vector<int>(c, 0));

	for(int i = 0; i < r; ++i)
	{
		for(int j = 0; j < c; ++j)
		{
			int current_match = int(left[i] == right[j]);
			int col_minus = (j > 0) ? lcs_table[i][j-1]: 0;
			int row_minus = (i > 0) ? lcs_table[i-1][j]: 0;
			int row_minus_col_minus = current_match + (i > 0 && j > 0 ? lcs_table[i-1][j-1]: 0);
			lcs_table[i][j] = max(col_minus, max(row_minus, row_minus_col_minus));
			}
	}
	return lcs_table[r-1][c-1];
}
```

Interestingly we can actually traverse this 2D array of the LCS backwards int order to identify the actual matching, addition, and subtraction characters. With this traverse we can extend this to git diff. Instead of working on a char basis, we can simply use lines.

```cpp
#pragma once

#include <algorithm>
#include <iostream>
#include <sstream>
#include <string>
#include <fstream>
#include <vector>

using namespace std;

namespace ab
{
vector<string> tokenize(ifstream& input)
{
  vector<string> tokens{};
  string line{};
  while(getline(input, line))
  {
    tokens.push_back(line);
  }
  return tokens;
}

vector<vector<int>> lcs(vector<string> const& tokens1, vector<string> const& tokens2)
{
  auto rows = tokens1.size();
  auto cols = tokens2.size();
  vector<vector<int>> lcs_table(rows, vector<int>(cols, 0));

  for(auto i = 0; i < rows; ++i)
  {
    for(auto j = 0; j < cols; ++j)
    {
      auto curr_matches = tokens1[i] == tokens2[j] ? 1 : 0;
      auto max_lcs = curr_matches + ((i > 0 && j > 0) ? lcs_table[i - 1][j - 1] : 0);
      max_lcs = max(max_lcs, (i > 0) ? lcs_table[i - 1][j] : 0);
      max_lcs = max(max_lcs, (j > 0) ? lcs_table[i][j - 1] : 0);
      lcs_table[i][j] = max_lcs;
    }
  }
  return lcs_table;
}

void file_diff(vector<vector<int>> const& lcs_table, vector<string> const& tokens1, vector<string> const& tokens2, ostream& output)
{
  assert(lcs_table.size() == tokens1.size());
  assert(lcs_table.size() && lcs_table[0].size() == tokens2.size());
  int rows = tokens1.size() - 1;
  int cols = tokens2.size() - 1;
  int i = rows, j = cols;
  vector<pair<char, string>> lines{};

  while(i >= 0 || j >= 0)
  {
    auto curr_matches = tokens1[i] == tokens2[j] ? 1 : 0;
    auto up_one_left_one = (i > 0 && j > 0) ? curr_matches + lcs_table[i - 1][j - 1] : 0;
    auto up_one = (i > 0) ? lcs_table[i - 1][j] : 0;
    auto left_one = (j > 0) ? lcs_table[i][j - 1] : 0;

    if((i == 0 && j == 0) || (up_one_left_one > up_one && up_one_left_one > left_one))
    {
      if(curr_matches)
      {
        lines.emplace_back(' ', tokens1[i]);
      }
      else
      {
        lines.emplace_back('-', tokens1[i]);
        lines.emplace_back('+', tokens2[j]);
      }
      --i, --j;
    }
    else if(up_one > left_one)
    {
      lines.emplace_back('-', tokens1[i]);
      --i;
    }
    else
    {
      lines.emplace_back('+', tokens2[j]);
      --j;
    }
  }

  reverse(begin(lines), end(lines));
  for(auto& [prefix, line]: lines)
  {
    output << prefix << " " << line << endl;
  }
}
```

It works alright I suppose. Implementing git diff for the whole repo is a bit more complicated. In order to do that, you have to hash the contents of the repo in both commit states. If you hash the contents of a file and directories, you can identify which files of the repo change between 2 commits, and selectively run the file diff on the changed files. That's for next time.