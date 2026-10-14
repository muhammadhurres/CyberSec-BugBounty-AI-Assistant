import gradio as gr

def generate_security_response(user_message, system_prompt):
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_message}
    ]
    
    text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
    model_inputs = tokenizer([text], return_tensors="pt").to("cuda")
    
    with torch.no_grad():
        generated_ids = model.generate(
            **model_inputs, 
            max_new_tokens=512,
            temperature=0.6,
            top_p=0.9
        )
        
    generated_ids = [
        output_ids[len(input_ids):] 
        for input_ids, output_ids in zip(model_inputs.input_ids, generated_ids)
    ]
    
    response = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]
    return response

default_system = (
    "You are an expert Cybersecurity Researcher and Bug Bounty Advisor. "
    "Explain security concepts clearly, analyze vulnerabilities, and write secure defensive code patches."
)

demo = gr.Interface(
    fn=generate_security_response,
    inputs=[
        gr.Textbox(lines=4, placeholder="Paste code or ask about XSS, SQLi, Buffer Overflows...", label="Bug Bounty / Security Query"),
        gr.Textbox(value=default_system, lines=2, label="System Persona")
    ],
    outputs=gr.Textbox(lines=12, label="Defensive Security Response"),
    title="🛡️ Cybersecurity & Bug Bounty AI Assistant"
)

# This will generate a local interface view AND a public shareable URL link
demo.launch(share=True)
