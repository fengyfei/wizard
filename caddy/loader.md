# Loader

## parser

![parser overview](./images/parser_overview.svg)

### 词法分析

词法分析代码如下：

```go
func allTokens(input io.Reader) ([]Token, error) {
	l := new(lexer)
	err := l.load(input)
	if err != nil {
		return nil, err
	}
	var tokens []Token
	for l.next() {
		tokens = append(tokens, l.token)
	}
	return tokens, nil
}
```

关键方法为 next():

```go
func (l *lexer) next() bool {
	var val []rune
	var comment, quoted, escaped bool

	makeToken := func() bool {
		l.token.Text = string(val)
		return true
	}

	for {
		ch, _, err := l.reader.ReadRune()
		if err != nil {
			if len(val) > 0 {
				return makeToken()
			}
			if err == io.EOF {
				return false
			}
			panic(err)
		}

		if quoted {
			if !escaped {
				if ch == '\\' {
					escaped = true
					continue
				} else if ch == '"' {
					quoted = false
					return makeToken()
				}
			}
			if ch == '\n' {
				l.line++
			}
			if escaped {
				// only escape quotes
				if ch != '"' {
					val = append(val, '\\')
				}
			}
			val = append(val, ch)
			escaped = false
			continue
		}

		if unicode.IsSpace(ch) {
			if ch == '\r' {
				continue
			}
			if ch == '\n' {
				l.line++
				comment = false
			}
			if len(val) > 0 {
				return makeToken()
			}
			continue
		}

		if ch == '#' {
			comment = true
		}

		if comment {
			continue
		}

		if len(val) == 0 {
			l.token = Token{Line: l.line}
			if ch == '"' {
				quoted = true
				continue
			}
		}

		val = append(val, ch)
	}
}
```

### executeDirectives

```go
func executeDirectives(inst *Instance, filename string,
	directives []string, sblocks []caddyfile.ServerBlock, justValidate bool) error {
	// map of server block ID to map of directive name to whatever.
	storages := make(map[int]map[string]interface{})

	// 保证指令的执行顺序
	for _, dir := range directives {
		for i, sb := range sblocks {
			var once sync.Once

			// ServerBlock 的 storage 不存在
			if _, ok := storages[i]; !ok {
				storages[i] = make(map[string]interface{})
			}

			for j, key := range sb.Keys {
				// 执行 ServerBlock 的指令
				if tokens, ok := sb.Tokens[dir]; ok {
					// 创建 Controller
					controller := &Controller{
						instance:  inst,
						Key:       key,
						Dispenser: caddyfile.NewDispenserTokens(filename, tokens),
						OncePerServerBlock: func(f func() error) error {
							var err error
							once.Do(func() {
								err = f()
							})
							return err
						},
						ServerBlockIndex:    i,
						ServerBlockKeyIndex: j,
						ServerBlockKeys:     sb.Keys,
						ServerBlockStorage:  storages[i][dir],
					}

					setup, err := DirectiveAction(inst.serverType, dir)
					if err != nil {
						return err
					}

					err = setup(controller)
					if err != nil {
						return err
					}

					storages[i][dir] = controller.ServerBlockStorage // persist for this server block
				}
			}
		}

		if !justValidate {
			// See if there are any callbacks to execute after this directive
			if allCallbacks, ok := parsingCallbacks[inst.serverType]; ok {
				callbacks := allCallbacks[dir]
				for _, callback := range callbacks {
					if err := callback(inst.context); err != nil {
						return err
					}
				}
			}
		}
	}

	return nil
}
```

- DirectiveAction

```go
func DirectiveAction(serverType, dir string) (SetupFunc, error) {
	// 查找 server type 类型插件集
	if stypePlugins, ok := plugins[serverType]; ok {
		// 查找 server type 下指令插件
		if plugin, ok := stypePlugins[dir]; ok {
			return plugin.Action, nil
		}
	}
	if genericPlugins, ok := plugins[""]; ok {
		if plugin, ok := genericPlugins[dir]; ok {
			return plugin.Action, nil
		}
	}
	return nil, fmt.Errorf("no action found for directive '%s' with server type '%s' (missing a plugin?)",
		dir, serverType)
}
```
