use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    pubkey::Pubkey,
    program_error::ProgramError,
    program_pack::{IsInitialized, Pack, Sealed},
    sysvar::{rent::Rent, Sysvar},
    program::invoke,
    program::invoke_signed,
    program::invoke_signed_unchecked,
    program_pack::Pack,
    system_instruction,
};

use spl_token::{
    state::Account as TokenAccount,
    instruction::transfer,
    ID as TOKEN_PROGRAM_ID,
};

#[derive(Clone, Debug, Default, PartialEq)]
pub struct TokenData {
    pub is_initialized: bool,
    pub burn_tax_wallet: Pubkey,
    pub excluded_wallets: [Pubkey; 3],
}

impl Sealed for TokenData {}

impl IsInitialized for TokenData {
    fn is_initialized(&self) -> bool {
        self.is_initialized
    }
}

impl Pack for TokenData {
    const LEN: usize = 1 + 32 + (32 * 3);
    fn pack_into_slice(&self, dst: &mut [u8]) {
        let output = array_mut_ref![dst, 0, TokenData::LEN];
        let (
            is_initialized_dst,
            burn_tax_wallet_dst,
            excluded_wallets_dst,
        ) = mut_array_refs![output, 1, 32, 96];
        is_initialized_dst[0] = self.is_initialized as u8;
        burn_tax_wallet_dst.copy_from_slice(self.burn_tax_wallet.as_ref());
        for i in 0..3 {
            excluded_wallets_dst[i * 32..(i + 1) * 32].copy_from_slice(self.excluded_wallets[i].as_ref());
        }
    }
    fn unpack_from_slice(src: &[u8]) -> Result<Self, ProgramError> {
        let input = array_ref![src, 0, TokenData::LEN];
        let (
            is_initialized_src,
            burn_tax_wallet_src,
            excluded_wallets_src,
        ) = array_refs![input, 1, 32, 96];
        let is_initialized = is_initialized_src[0] != 0;
        let burn_tax_wallet = Pubkey::new_from_array(*burn_tax_wallet_src);
        let mut excluded_wallets = [Pubkey::default(); 3];
        for i in 0..3 {
            excluded_wallets[i] = Pubkey::new_from_array(*array_ref![excluded_wallets_src, i * 32, 32]);
        }
        Ok(TokenData {
            is_initialized,
            burn_tax_wallet,
            excluded_wallets,
        })
    }
}

entrypoint!(process_instruction);

fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let burn_tax_wallet_account = next_account_info(account_info_iter)?;
    let token_account = next_account_info(account_info_iter)?;

    let mut token_data = TokenData::unpack(&token_account.data.borrow())?;
    if !token_data.is_initialized {
        token_data.is_initialized = true;
        token_data.burn_tax_wallet = *burn_tax_wallet_account.key;
        TokenData::pack(token_data, &mut token_account.data.borrow_mut())?;
    } else {
        let source_account = next_account_info(account_info_iter)?;
        let destination_account = next_account_info(account_info_iter)?;
        let token_program = next_account_info(account_info_iter)?;
        if token_program.key != &TOKEN_PROGRAM_ID {
            return Err(ProgramError::IncorrectProgramId);
        }
        let amount = 0.00188 * 10000; // 0.188% tax
        let ix = transfer(
            &TOKEN_PROGRAM_ID,
            &source_account.key,
            &token_data.burn_tax_wallet,
            &source_account.key,
            &[],
            amount,
        )?;
        msg!("Transferring tax to burn wallet");
        invoke(
            &ix,
            &[
                source_account.clone(),
                burn_tax_wallet_account.clone(),
                token_program.clone(),
            ],
        )?;
    }

    Ok(())
}

// Define more functionality as needed...

