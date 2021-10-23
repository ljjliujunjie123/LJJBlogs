### 语法

### 源码

核心就这一个类，比较重要的还有两三个

```kotlin
public actual class Regex internal constructor(private val nativePattern: Pattern) : Serializable {
    
    public actual constructor(pattern: String) : this(Pattern.compile(pattern))
    
    public actual constructor(pattern: String, option: RegexOption) : this(Pattern.compile(pattern, ensureUnicodeCase(option.value)))
    
    public actual constructor(pattern: String, options: Set<RegexOption>) : this(Pattern.compile(pattern, ensureUnicodeCase(options.toInt())))

    public actual val pattern: String
        get() = nativePattern.pattern()

    private var _options: Set<RegexOption>? = null
    
    public actual val options: Set<RegexOption> get() = _options ?: fromInt<RegexOption>(nativePattern.flags()).also { _options = it }

    public actual infix fun matches(input: CharSequence): Boolean = nativePattern.matcher(input).matches()

    public actual fun containsMatchIn(input: CharSequence): Boolean = nativePattern.matcher(input).find()

    public actual fun find(input: CharSequence, startIndex: Int = 0): MatchResult? =
        nativePattern.matcher(input).findNext(startIndex, input)

    public actual fun findAll(input: CharSequence, startIndex: Int = 0): Sequence<MatchResult> {
        if (startIndex < 0 || startIndex > input.length) {
            throw IndexOutOfBoundsException("Start index out of bounds: $startIndex, input length: ${input.length}")
        }
        return generateSequence({ find(input, startIndex) }, MatchResult::next)
    }

    public actual fun matchEntire(input: CharSequence): MatchResult? = nativePattern.matcher(input).matchEntire(input)

    public actual fun replace(input: CharSequence, replacement: String): String = nativePattern.matcher(input).replaceAll(replacement)

    public actual fun replace(input: CharSequence, transform: (MatchResult) -> CharSequence): String {
        var match: MatchResult? = find(input) ?: return input.toString()

        var lastStart = 0
        val length = input.length
        val sb = StringBuilder(length)
        do {
            val foundMatch = match!!
            sb.append(input, lastStart, foundMatch.range.start)
            sb.append(transform(foundMatch))
            lastStart = foundMatch.range.endInclusive + 1
            match = foundMatch.next()
        } while (lastStart < length && match != null)

        if (lastStart < length) {
            sb.append(input, lastStart, length)
        }

        return sb.toString()
    }

    public actual fun replaceFirst(input: CharSequence, replacement: String): String =
        nativePattern.matcher(input).replaceFirst(replacement)

    public actual fun split(input: CharSequence, limit: Int = 0): List<String> {
        require(limit >= 0, { "Limit must be non-negative, but was $limit." })

        val matcher = nativePattern.matcher(input)
        if (!matcher.find() || limit == 1) return listOf(input.toString())

        val result = ArrayList<String>(if (limit > 0) limit.coerceAtMost(10) else 10)
        var lastStart = 0
        val lastSplit = limit - 1 // negative if there's no limit

        do {
            result.add(input.substring(lastStart, matcher.start()))
            lastStart = matcher.end()
            if (lastSplit >= 0 && result.size == lastSplit) break
        } while (matcher.find())

        result.add(input.substring(lastStart, input.length))

        return result
    }

    public override fun toString(): String = nativePattern.toString()

    public fun toPattern(): Pattern = nativePattern

    private fun writeReplace(): Any = Serialized(nativePattern.pattern(), nativePattern.flags())

    private class Serialized(val pattern: String, val flags: Int) : Serializable {
        companion object {
            private const val serialVersionUID: Long = 0L
        }

        private fun readResolve(): Any = Regex(Pattern.compile(pattern, flags))
    }

    actual companion object {

        public actual fun fromLiteral(literal: String): Regex = literal.toRegex(RegexOption.LITERAL)

        public actual fun escape(literal: String): String = Pattern.quote(literal)

        public actual fun escapeReplacement(literal: String): String = Matcher.quoteReplacement(literal)

        private fun ensureUnicodeCase(flags: Int) = if (flags and Pattern.CASE_INSENSITIVE != 0) flags or Pattern.UNICODE_CASE else flags
    }

}
```

### 常见例子

