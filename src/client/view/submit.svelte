<script>
  import { onMount, afterUpdate, createEventDispatcher, tick } from 'svelte'
  import { options, msg, lazy } from '../lib/stores'
  import request from '../lib/request'
  import emotFn from '../lib/emot'
  import Emotion from '../../../assets/svg/Emotion.svg'
  import Loading from '../../../assets/svg/Loading.svg'
  import Setting from '../../../assets/svg/Setting.svg'
  import Refresh from '../../../assets/svg/Refresh.svg'
  import { translate } from '../i18n'

  let D = $options
  const dispatch = createEventDispatcher()

  export let cancel = false,
    pid = '',
    rid = '',
    wordLimit = {
      nick: 0,
      mail: 0,
      site: 0,
      content: 0
    }

  // 普通变量
  const textStr = 'text'
  const nickStr = 'nick'
  const mailStr = 'mail'
  const siteStr = 'site'
  const contentStr = 'content'
  const mailReg = /^\w+([-.]\w+)*@\w+([-.]\w+)*(\.[a-z]{2,5})+$/

  // svelte变量
  let storage = localStorage.discuss
  let isEmot = false
  let emotIndex = 0
  let emotMaps = {}
  let emotAll = {}
  let textareaDOM
  let isPreview = false
  let isSend = false
  let isLegal = false
  const inputs = [
    {
      key: nickStr,
      locale: translate(nickStr),
      type: textStr
    },
    {
      key: mailStr,
      locale: translate(mailStr),
      type: 'e' + mailStr
    },
    {
      key: siteStr,
      locale: translate(siteStr),
      type: textStr
    }
  ]
  let metas = {
    nick: { value: '', is: false },
    mail: { value: '', is: false },
    site: { value: '', is: true },
    content: { value: '', is: false }
  }
  let contentHTML = ''
  let wordLimitContent = wordLimit.content
  let limitContentLen
  $: {
    wordLimitContent = wordLimit.content
    const dom = new DOMParser().parseFromString(contentHTML, 'text/html')
    limitContentLen = dom.body.textContent.length + dom.body.querySelectorAll('img').length
  }

  onMount(() => {
    initInfo()
    getEmot()
    onInput()
  })

  afterUpdate(() => {
    metasChange()
    $lazy()
  })

  function initInfo() {
    try {
      storage = JSON.parse(storage) || {}
      for (const [k, v] of Object.entries(storage)) {
        metas[k].value = v || ''
      }
    } catch (error) {
      storage = {}
    }
  }

  async function getEmot() {
    const emot = D.emotMaps
    if (/\.json$/.test(emot)) {
      emotMaps = await request({ url: emot, method: 'GET' })
    } else if (!emot) {
      emotMaps = emotFn(D.emotCDN)
    } else {
      emotMaps = emot
    }
    getEmotAll()
  }

  function getEmotAll() {
    try {
      for (const e in emotMaps) {
        const type = emotMaps[e].type
        if (type === textStr) continue
        const items = emotMaps[e].items
        emotAll = { ...emotAll, ...items }
      }
    } catch (error) {
      // eslint-disable-next-line no-console
      console.log(error)
    }
  }
  function parseEmot() {
    let content = metas.content.value
    const emots = []
    content.replace(/\[(.*?)\]/g, ($0, $1) => {
      emots.push($1)
    })

    for (const emot of emots) {
      const link = emotAll[emot]
      if (!link) continue
      const img = `<img class='D-comment-emot' src='${link}' alt='${emot}'/>`
      content = content.replace(`[${emot}]`, img)
    }
    contentHTML = content
  }

  function saveInfo() {
    for (const [k, v] of Object.entries(metas)) {
      storage[k] = v.value.trim()
    }
    localStorage.discuss = JSON.stringify(storage)
    // 重新解析表情
    parseEmot()
  }

  function onPreview() {
    isPreview = !isPreview
  }

  function onInput() {
    saveInfo()
    metasChange()
  }
  /**
   * @param {String} key 表情名(描述)
   * @param {String} value 表情值(内容或地址)
   * @param {String} type 表情类型(text or image)
   */
  function onClickEmot(key, value, type) {
    const content = metas.content.value

    // 获取输入框光标位置
    let cursorStart = textareaDOM.selectionStart
    let cursorEnd = textareaDOM.selectionEnd
    const Start = content.substring(0, cursorStart)
    const Ent = content.substring(cursorEnd)

    let range
    textareaDOM.focus()
    if (type === textStr) {
      metas.content.value = `${Start}${value}${Ent}`
      range = (Start + value).length
    } else {
      metas.content.value = `${Start}[${key}]${Ent}`
      range = (Start + key).length + 2
    }
    // 重新保存
    saveInfo()

    tick().then(() => {
      textareaDOM.setSelectionRange(range, range)
    })
  }

  // eslint-disable-next-line max-statements
  function metasChange() {
    try {
      const { nick, mail, site, content } = metas
      const { nick: nickWord, mail: mailWord, content: contentWord } = wordLimit
      const nickLen = nick.value.length
      const mailLen = mail.value.length

      // 昵称
      if ((nickWord === 0 && nickLen > 1) || (nickLen > 1 && nickLen <= nickWord)) {
        nick.is = true
      } else {
        nick.is = false
      }

      // 邮箱
      if ((mailWord === 0 && mailReg.test(mail.value)) || (mailLen <= mailWord && mailReg.test(mail.value))) {
        mail.is = true
      } else {
        mail.is = false
      }

      // 内容
      const dom = new DOMParser().parseFromString(contentHTML, 'text/html')
      const textContent = dom.body.textContent.length + dom.querySelectorAll('img').length
      if ((contentWord === 0 && textContent > 1) || (textContent > 1 && textContent <= contentWord)) {
        content.is = true
      } else {
        content.is = false
      }
      metas = metas
      isLegal = nick.is && mail.is && site.is && content.is
    } catch (error) {
      // eslint-disable-next-line no-console
      console.error(error)
    }
  }

  // eslint-disable-next-line max-statements
  async function onSend() {
    try {
      if (!isSend && !isLegal) return
      const comment = {
        type: 'COMMIT_COMMENT',
        nick: metas.nick.value,
        mail: metas.mail.value,
        site: metas.site.value,
        content: contentHTML,
        path: D.path,
        pid,
        rid
      }
      isSend = true

      const token = localStorage.DToken
      if (token) comment.token = token
      const result = await request({
        url: D.serverURLs,
        data: comment
      })

      if (!result.data && result.msg.includes('login')) {
        $msg({ type: 'error', text: translate('pleaseLogin') })
      }

      if (Array.isArray(result.data)) {
        if (!result.data.length) $msg({ type: 'success', duration: 5000, text: translate('commentsAudit') })

        dispatch('submitComment', { comment: result.data, pid })
        metas.content.value = ''
        saveInfo()
        isPreview = false
      }
    } catch (error) {
      // eslint-disable-next-line no-console
      console.error('Comment failure:', error)
      $msg({ type: 'error', text: translate('sendError') })
    } finally {
      isSend = false
    }
  }
</script>

<div class="D-submit">
  <div class="D-input">
    {#each inputs as i}
      <input
        bind:value={metas[i.key].value}
        class={metas[i.key].is ? '' : 'D-error'}
        name={i.key}
        placeholder={i.locale}
        on:input={(e) => (e.target.type = i.type)}
        on:input={onInput}
      />
    {/each}
    <textarea
      name={contentStr}
      class="D-input-content {metas.content.is ? '' : 'D-error'}"
      bind:value={metas.content.value}
      placeholder={D.ph}
      on:input={onInput}
      bind:this={textareaDOM}
    />
    {#if wordLimitContent}
      <span class="D-text-number">
        {limitContentLen}
        {#if wordLimitContent}
          <span class={limitContentLen > wordLimitContent && 'D-text-number-illegal'}>{'/ ' + wordLimitContent} </span>
        {/if}
      </span>
    {/if}
  </div>
  <div class="D-actions D-select-none">
    <div class="D-actions-left">
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <div class="D-emot-btn" on:click={() => (isEmot = !isEmot)}>
        <Emotion />
      </div>
      {#if !cancel}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div class="D-setting-btn" on:click={() => dispatch('onSetting')}><Setting /></div>
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div class="D-refresh-btn" on:click={() => dispatch('onRefresh')}><Refresh /></div>
      {/if}
    </div>

    <div class="D-actions-right">
      <!-- 取消回复 -->
      {#if cancel}
        <button class="D-cancel D-btn D-btn-main" on:click={() => dispatch('onCancel', true)}
          >{translate('cancel')}</button
        >
      {/if}

      <button
        on:click={onPreview}
        class="D-cancel D-btn D-btn-main {/* 没有内容则禁用预览按钮 */ !metas.content.value.length && 'D-disabled'}"
        >{translate('preview')}</button
      ><button class="D-send D-btn D-btn-main" on:click={onSend} disabled={isSend || !isLegal}>
        {#if isSend && isLegal}
          <Loading />
        {:else}
          {translate('send')}
        {/if}
      </button>
    </div>
  </div>
  {#if isPreview}
    <div class="D-preview">{@html contentHTML}</div>
  {/if}
  {#if isEmot}
    <div class="D-emot">
      {#each Object.entries(emotMaps) as [emotKey, emotValue], index}
        <ul class="D-emot-items {index === emotIndex ? 'D-emot-items-active' : ''}">
          {#each Object.entries(emotValue.items) as [iKey, iValue]}
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <li class="D-emot-item" on:click={onClickEmot(iKey, iValue, emotValue.type)}>
              {#if emotValue.type === 'text'}
                <span title={iKey}>{iValue}</span>
              {:else}
                <img class="D-comment-emot" src={D.imgLoading} d-src={iValue} alt={iKey} title={iKey} />
              {/if}
            </li>
          {/each}
        </ul>
      {/each}

      <div class="D-emot-packages">
        {#each Object.entries(emotMaps) as [emotKey, emotValue], index}
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <span on:click={() => (emotIndex = index)} class={index === emotIndex ? 'D-emot-package-active' : ''}
            >{@html emotKey}</span
          >
        {/each}
      </div>
    </div>
  {/if}
</div>

<style lang="scss">
  /* submit */
  .D-submit {
    margin: 10px 0;
    padding: 10px;
    border-radius: 8px;
    border: solid 1px var(--D-Centre-Color);

    &:hover {
      border-color: var(--D-Height-Color);
    }
    .D-input .D-error {
      border-radius: 6px;
      border-color: var(--D-main-Color);
      background: rgba(244, 100, 95, 0.1);
    }
  }
  .D-input {
    position: relative;
    input {
      padding: 6px;
      width: calc((100% - 1rem) / 3);
      outline: none;
      border-bottom: dashed 1px var(--D-Centre-Color);

      + input {
        margin-left: 0.5rem;
      }
    }
    * {
      color: currentColor;
      border: none;
      background: transparent;
      box-sizing: border-box;

      &:focus {
        border-radius: 8px;
        background: rgba(153, 153, 153, 0.08);
      }
    }
    *:hover {
      border-color: var(--D-Height-Color);
      transition: all 0.5s;
    }

    .D-input-content {
      margin: 10px 0 0;
      resize: vertical;
      width: 100%;
      min-height: 140px;
      max-height: 400px;
      outline: none;
      font-family: inherit;
      transition: none;
    }

    .D-text-number {
      position: absolute;
      color: #999;
      right: 14px;
      bottom: 6px;
      font-size: 12px;
    }

    .D-text-number-illegal {
      color: red;
    }
    .D-error {
      border-radius: 6px;
      border-color: var(--D-main-Color);
      background: rgba(244, 100, 95, 0.1);
    }
  }

  .D-actions {
    margin: 10px 0 0;

    .D-actions-left {
      display: flex;
    }

    .D-actions-right {
      display: flex;
      align-items: center;

      .D-btn {
        margin-left: 4px;
      }
    }
  }

  .D-actions,
  .D-emot-btn,
  .D-setting-btn,
  .D-refresh-btn {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .D-setting-btn,
  .D-refresh-btn {
    width: 18px;
    cursor: pointer;
    margin-left: 6px;
  }

  .D-send {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  /* emot */
  .D-emot {
    top: 30px;
    width: 100%;
    margin-top: 10px;
    border: 1px solid var(--D-Low-Color);
    border-radius: 4px;
  }

  .D-emot-items {
    display: none;
    height: 180px;
    min-height: 100px;
    max-height: 200px;
    resize: vertical;
    padding: 10px;
    margin: 0;
    overflow-x: hidden;
    user-select: none;
  }

  .D-emot-items-active {
    display: block;
  }

  .D-emot-item {
    font-size: 20px;
    list-style-type: none;
    padding: 5px 10px;
    border-radius: 5px;
    display: inline-block;
    line-height: 14px;
    margin: 0 10px 12px 0;
    cursor: pointer;
    transition: 0.3s;

    &:hover {
      background: var(--D-Low-Color);
      box-shadow: 0 2px 2px 0 rgb(0 0 0 / 14%), 0 3px 1px -2px rgb(0 0 0 / 20%), 0 1px 5px 0 rgb(0 0 0 / 12%);
    }
  }

  .D-emot-packages {
    padding: 0;
    font-size: 0;
    border-top: solid 1px var(--D-Low-Color);

    span {
      display: inline-block;
      line-height: 30px;
      font-size: 14px;
      padding: 0 10px;
      cursor: pointer;

      :global(img) {
        width: 20px;
        position: relative;
        top: 5px;
      }

      &:nth-child(1) {
        border-radius: 0 0 0 3px;
      }
    }
  }

  .D-emot-package-active {
    background: var(--D-Low-Color);
  }

  /* preview */

  .D-preview {
    padding: 10px;
    overflow-x: auto;
    min-height: 1.375rem /* 22/16 */;
    margin: 10px 0;
    border: 1px solid #dcdfe6;
    border-radius: 4px;
  }

  @media screen and (max-width: 500px) {
    .D-input {
      display: flex;
      flex-direction: column;
    }
    .D-input input {
      width: 100%;
    }
    .D-input input + input {
      margin-top: 4px;
      margin-left: 0;
    }
  }
</style>
