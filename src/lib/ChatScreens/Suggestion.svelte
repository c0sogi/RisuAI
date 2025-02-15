<script lang="ts">
	import { requestChatData } from "src/ts/process/request";
    import { doingChat, type OpenAIChat } from "../../ts/process/index";
    import { DataBase, setDatabase, type character, type Message, type groupChat, type Database } from "../../ts/storage/database";
    import { selectedCharID } from "../../ts/stores";
    import { translate } from "src/ts/translator/translator";
    import { CopyIcon, LanguagesIcon, RefreshCcwIcon } from "lucide-svelte";
    import { alertConfirm } from "src/ts/alert";
    import { language } from "src/lang";
    import { replacePlaceholders } from "../../ts/util";
    import { onDestroy } from 'svelte';
    import { processScript } from "src/ts/process/scripts";
    import { get } from "svelte/store";
    import { ParseMarkdown } from "src/ts/parser";

    export let send: () => any;
    export let messageInput:(string:string) => any;
    let suggestMessages:string[] = $DataBase.characters[$selectedCharID]?.chats[$DataBase.characters[$selectedCharID].chatPage]?.suggestMessages
    let suggestMessagesTranslated:string[]
    let toggleTranslate:boolean = $DataBase.autoTranslate
    let progress:boolean;
    let progressChatPage=-1;
    let abortController:AbortController;
    let chatPage:number
    $: {
        $selectedCharID
        //FIXME add selectedChatPage for optimize render
        chatPage = $DataBase.characters[$selectedCharID].chatPage
        updateSuggestions()
    }

    const updateSuggestions = () => {
        if($selectedCharID > -1 && !$doingChat) {
            if(progressChatPage > 0 && progressChatPage != chatPage){
                progress=false
                abortController?.abort()
            }
            let currentChar = $DataBase.characters[$selectedCharID];
            suggestMessages = currentChar?.chats[currentChar.chatPage].suggestMessages
        }
    }
    

    const unsub = doingChat.subscribe((v) => {
        if(v) {
            progress=false
            abortController?.abort()
            suggestMessages = []
        }
        if(!v && $selectedCharID > -1 && (!suggestMessages || suggestMessages.length === 0) && !progress){
            let currentChar:character|groupChat = $DataBase.characters[$selectedCharID];
            let messages:Message[] = []
            
            if(currentChar.type !== 'group'){
                const firstMsg:string = currentChar.firstMsgIndex === -1 ? currentChar.firstMessage : currentChar.alternateGreetings[currentChar.firstMsgIndex]
                messages.push({
                    role: 'char',
                    data: processScript(currentChar,
                        replacePlaceholders(firstMsg, currentChar.name),
                    'editprocess')
                })
            }
            messages = [...messages, ...currentChar.chats[currentChar.chatPage].message];
            let lastMessages:Message[] = messages.slice(Math.max(messages.length - 10, 0));
            if(lastMessages.length === 0)
                return
            const promptbody:OpenAIChat[] = [
            {
                role:'system',
                content: replacePlaceholders($DataBase.autoSuggestPrompt, currentChar.name)
            }
            ,{
                role: 'user', 
                content: lastMessages.map(b=>(b.role==='char'? currentChar.name : $DataBase.username)+":"+b.data).reduce((a,b)=>a+','+b)
            }
            ]

            progress = true
            progressChatPage = chatPage
            abortController = new AbortController()
            requestChatData({
                formated: promptbody,
                bias: {},
                currentChar : currentChar as character
            }, 'submodel', abortController.signal).then(rq2=>{
                if(rq2.type !== 'fail' && rq2.type !== 'streaming' && progress){
                    var suggestMessagesNew = rq2.result.split('\n').filter(msg => msg.startsWith('-')).map(msg => msg.replace('-','').trim())
                    const db:Database = get(DataBase);
                    db.characters[$selectedCharID].chats[currentChar.chatPage].suggestMessages = suggestMessagesNew
                    setDatabase(db)
                    suggestMessages = suggestMessagesNew
                }
                progress = false
            })
            }
    })

    const translateSuggest = async (toggle, messages)=>{
        if(toggle && messages && messages.length > 0) {
            suggestMessagesTranslated = []
            for(let i = 0; i < suggestMessages.length; i++){
                let msg = suggestMessages[i]
                let translated = await translate(msg, false)
                suggestMessagesTranslated[i] = translated
            }
        }
    }

    onDestroy(unsub)

    $: {translateSuggest(toggleTranslate, suggestMessages)}
</script>

<div class="ml-4 flex flex-wrap">
    {#if progress}
        <div class="flex bg-gray-500 p-2 rounded-lg items-center">
            <div class="loadmove mx-2"/>
            <div>{language.creatingSuggestions}</div>
        </div>        
    {:else if !$doingChat}
        {#if $DataBase.translator !== ''}
            <div class="flex mr-2 mb-2">
                <button class={"bg-gray-500 hover:bg-gray-700 font-bold py-2 px-4 rounded " + (toggleTranslate ? 'text-green-500' : 'text-white')}
                    on:click={() => {
                        toggleTranslate = !toggleTranslate
                    }}
                >
                    <LanguagesIcon/>
                </button>
            </div>    
        {/if}
        

        <div class="flex mr-2 mb-2">
            <button class="bg-gray-500 hover:bg-gray-700 font-bold py-2 px-4 rounded text-white"
                on:click={() => {
                    alertConfirm(language.askReRollAutoSuggestions).then((result) => {
                        if(result) {
                            suggestMessages = []
                            doingChat.set(true)
                            doingChat.set(false)        
                        }
                    })
                }}
            >
                <RefreshCcwIcon/>
            </button>
        </div>
        {#each suggestMessages??[] as suggest, i}
            <div class="flex mr-2 mb-2">
                <button class="bg-gray-500 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded" on:click={() => {
                    suggestMessages = []
                    messageInput(suggest)
                    send()
                }}>
                {#await ParseMarkdown(($DataBase.translator !== '' && toggleTranslate && suggestMessagesTranslated && suggestMessagesTranslated.length > 0) ? suggestMessagesTranslated[i]??suggest : suggest) then md}
                    {@html md}
                {/await}
                </button>
                <button class="bg-gray-500 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded ml-1" on:click={() => {
                    messageInput(suggest)
                }}>
                    <CopyIcon/>
                </button>
            </div>
        {/each}
        
    {/if}
</div>

<style>
    
    .loadmove {
        animation: spin 1s linear infinite;
        border-radius: 50%;
        border: 0.4rem solid rgba(0,0,0,0);
        width: 1rem;
        height: 1rem;
        border-top: 0.4rem solid white;
        border-left: 0.4rem solid white;
    }

    @keyframes spin {
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
    }
</style>

